## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Swapchain 

Okay, we have a surface (and the OS specific stuff is definitely over), how do we draw on it? Well, the thing is, we usually do not want to draw onto
that which is visible. A classical way is to have a frontbuffer and a backbuffer. We show the frontbuffer on the screen and draw on the backbuffer.
When we're done drawing, we switch. We show the picture we have just drawn, and draw something new in the other one (which is currently not shown on
the screen). This set of {frontbuffer, backbuffer} is a simple example of a swapchain. We could use three pictures instead ... 

Swapchains are another extension, this time a device extension, not an instance extension. Therefore, it is the `DeviceCreateInfo` that we adjust: 
```rust
     let device_extension_name_pointers: Vec<*const i8> =
        vec![ash::extensions::khr::Swapchain::name().as_ptr()];
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers);
```
Before actually creating a swapchain, let us query some information about the surface we are dealing with; mainly to get a better idea about how to
fill the necessary structs. 
```rust
    let surface_capabilities = unsafe {
        surface_loader.get_physical_device_surface_capabilities(physical_device, surface)
    };
    let surface_present_modes = unsafe {
        surface_loader.get_physical_device_surface_present_modes(physical_device, surface)
    };
    let surface_formats =
        unsafe { surface_loader.get_physical_device_surface_formats(physical_device, surface) };
    dbg!(&surface_capabilities);
    dbg!(&surface_present_modes);
    dbg!(&surface_formats);
```

Let us actually create a swapchain: 
```rust
    let queuefamilies = [qfamindices.0];
    let swapchain_create_info = vk::SwapchainCreateInfoKHR::builder()
        .surface(surface)
        .min_image_count(
            3.max(surface_capabilities.min_image_count)
                .min(surface_capabilities.max_image_count),
        )
        .image_format(surface_formats.first().unwrap().format)
        .image_color_space(surface_formats.first().unwrap().color_space)
        .image_extent(surface_capabilities.current_extent)
        .image_array_layers(1)
        .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
        .image_sharing_mode(vk::SharingMode::EXCLUSIVE)
        .queue_family_indices(&queuefamilies)
        .pre_transform(surface_capabilities.current_transform)
        .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
        .present_mode(vk::PresentModeKHR::FIFO);
    let swapchain_loader = ash::extensions::khr::Swapchain::new(&instance, &logical_device);
    let swapchain = unsafe { swapchain_loader.create_swapchain(&swapchain_create_info, None)? };
```
That is, we create a swapchain ... for the surface we have created (makes sense), with 3 images (frontbuffer, backbuffer, backestbuffer?), but at
least the minimal required amount and at most the maximal possible amount, with a certain image format and colour space, which for me will be 
```rust
    SurfaceFormatKHR {
        format: B8G8R8A8_UNORM,
        color_space: SRGB_NONLINEAR,
    },
```
(if I'm just copying it from the previous debug output). Here the format means 8 bit each for blue, green, red, alpha (in that order), in an unsigned,
normalized format, that means in the range [0,1]. And colour (in what we will want to draw on the screen) is encoded as sRGB, that means doubling the
values describing some gray colour fits well with perceived doubling of brightness, but linearly interpolating colours does not work too well. (sRGB
is plain linear RGB plus gamma correction.) 

Moving on: We need the size of the frame (here 800x600 pixels, because that seems to be the default if we don't tell winit anything else — but it doesn't
really matter, because we just take the size ("extent") that we are currently using (see `surface_capabilities`)). 

I have one screen, not a fancy set of
VR glasses, so I need one image at a time (and I think this is what `image_array_layers` is good for). We want to use the results from this swapchain as
`COLOR_ATTACHMENT`s. We will access images from only one queue family at a time, we don't have to share access (`SharingMode::EXCLUSIVE`), and the queue
family is that for graphics. 

As transform we could specify something like "rotated clockwise by a quarter circle" or "mirrored horizontally", but the
surface capabilities tell me that nothing else than an identity transform would work in my case anyway. Let us just use the current one. 

Composite alpha has to do with how the application window is supposed to interact with other windows. We are not going for any transparency effects,
let's just set it to OPAQUE. Note: The transparency here refers only to transparency of the window itself, it has absolutely nothing to do with the
transparency of objects we will later draw inside the window. (If we ever proceed far enough to actually draw something, that is.) 

Finally, a presentation mode. If we have front- and backbuffer only, this is not an interesting question: at some point we switch their content. Now
we have three such things to draw on. Assume, the screen is busy showing no. 1, while we are fast and prepare 2 and 3, before it's time to use a new
picture. Which one should be chosen? We pick the FIFO mode: Show every one that was prepared, in order. What else is there? If you're interested, read
the spec (entry: [vkPresentModeKHR](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPresentModeKHR.html)), there's a better description than I could give here. 

With the CreateInfo struct filled (how to know which fields to fill? — As usual: either starting with a
`vk::SwapchainCreateInfoKHR{..Default::default()}` and glimpsing at its `dbg!`-output, or following along with the list of methods in the documentation of
ash or with the list in the Vulkan spec), we then first load the extension and call a specific creation function. And, by the way: what we create with
a `create_...`-function, we should later (at the end of the program) destroy again: 
```rust
        swapchain_loader.destroy_swapchain(swapchain, None);
```
Now we have a swapchain. Having a swapchain means having some images (probably three, the way we've set this up). Let's get them: 
```rust
    let swapchain_images = unsafe { swapchain_loader.get_swapchain_images(swapchain)? };
```
This is a Vec of`vkImages`. 
An image is not yot much. In particular, it is not accessed directly for reading or writing image data. Instead, we need some ImageView. 
```rust
    let mut swapchain_imageviews = Vec::with_capacity(swapchain_images.len());
    for image in &swapchain_images {
        let subresource_range = vk::ImageSubresourceRange::builder()
            .aspect_mask(vk::ImageAspectFlags::COLOR)
            .base_mip_level(0)
            .level_count(1)
            .base_array_layer(0)
            .layer_count(1);
        let imageview_create_info = vk::ImageViewCreateInfo::builder()
            .image(*image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::B8G8R8A8_UNORM)
            .subresource_range(*subresource_range);
        let imageview = unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;
        swapchain_imageviews.push(imageview);
    }
```
Here we create imageviews for each of these images. We have to say: for which image, what type it is (2D image, or 1D, 3D, a cube map?), the format
(try changing this; if it's something different from the format given to the swapchain create info, the validation layers will complain) and a
subresource range (saying that we're interested in the Color (not depth info) of this image, and have one level starting at number 0 for`mip_levels` 
and `array_layers` each, because we can ignore them for the moment. 

And again: explicit creation means explicit destruction at the end: 
```rust
        for iv in &swapchain_imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
```
The current program: 
```rust
use ash::version::DeviceV1_0;
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;
use ash::vk;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;

    let entry = ash::Entry::new()?;
    let enginename = std::ffi::CString::new("UnknownGameEngine").unwrap();
    let appname = std::ffi::CString::new("The Black Window").unwrap();
    let app_info = vk::ApplicationInfo::builder()
        .application_name(&appname)
        .application_version(vk::make_version(0, 0, 1))
        .engine_name(&enginename)
        .engine_version(vk::make_version(0, 42, 0))
        .api_version(vk::make_version(1, 0, 106));
    let layer_names: Vec<std::ffi::CString> =
        vec![std::ffi::CString::new("VK_LAYER_KHRONOS_validation").unwrap()];
    let layer_name_pointers: Vec<*const i8> = layer_names
        .iter()
        .map(|layer_name| layer_name.as_ptr())
        .collect();
    let extension_name_pointers: Vec<*const i8> = vec![
        ash::extensions::ext::DebugUtils::name().as_ptr(),
        ash::extensions::khr::Surface::name().as_ptr(),
        ash::extensions::khr::XlibSurface::name().as_ptr(),
    ];
    let mut debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
        .message_severity(
            vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
                | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
                | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
                | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
        )
        .message_type(
            vk::DebugUtilsMessageTypeFlagsEXT::GENERAL
                | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE
                | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION,
        )
        .pfn_user_callback(Some(vulkan_debug_utils_callback));

    let instance_create_info = vk::InstanceCreateInfo::builder()
        .push_next(&mut debugcreateinfo)
        .application_info(&app_info)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_extension_names(&extension_name_pointers);
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };
    let debug_utils = ash::extensions::ext::DebugUtils::new(&entry, &instance);
    let utils_messenger =
        unsafe { debug_utils.create_debug_utils_messenger(&debugcreateinfo, None)? };

    use winit::platform::unix::WindowExtUnix;
    let x11_display = window.xlib_display().unwrap();
    let x11_window = window.xlib_window().unwrap();
    let x11_create_info = vk::XlibSurfaceCreateInfoKHR::builder()
        .window(x11_window)
        .dpy(x11_display as *mut vk::Display);
    let xlib_surface_loader = ash::extensions::khr::XlibSurface::new(&entry, &instance);
    let surface = unsafe { xlib_surface_loader.create_xlib_surface(&x11_create_info, None) }?;
    let surface_loader = ash::extensions::khr::Surface::new(&entry, &instance);

    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    let (physical_device, physical_device_properties) = {
        let mut chosen = None;
        for p in phys_devs {
            let properties = unsafe { instance.get_physical_device_properties(p) };
            if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
                chosen = Some((p, properties));
            }
        }
        chosen.unwrap()
    };
    let queuefamilyproperties =
        unsafe { instance.get_physical_device_queue_family_properties(physical_device) };
    let qfamindices = {
        let mut found_graphics_q_index = None;
        let mut found_transfer_q_index = None;
        for (index, qfam) in queuefamilyproperties.iter().enumerate() {
            if qfam.queue_count > 0
                && qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
                && unsafe {
                    surface_loader
                        .get_physical_device_surface_support(physical_device, index as u32, surface)?
                }
            {
                found_graphics_q_index = Some(index as u32);
            }
            if qfam.queue_count > 0 && qfam.queue_flags.contains(vk::QueueFlags::TRANSFER) {
                if found_transfer_q_index.is_none()
                    || !qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
                {
                    found_transfer_q_index = Some(index as u32);
                }
            }
        }
        (
            found_graphics_q_index.unwrap(),
            found_transfer_q_index.unwrap(),
        )
    };
    let priorities = [1.0f32];
    let queue_infos = [
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(qfamindices.0)
            .queue_priorities(&priorities)
            .build(),
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(qfamindices.1)
            .queue_priorities(&priorities)
            .build(),
    ];
    let device_extension_name_pointers: Vec<*const i8> =
        vec![ash::extensions::khr::Swapchain::name().as_ptr()];
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers);
    let logical_device =
        unsafe { instance.create_device(physical_device, &device_create_info, None)? };
    let graphics_queue = unsafe { logical_device.get_device_queue(qfamindices.0, 0) };
    let transfer_queue = unsafe { logical_device.get_device_queue(qfamindices.1, 0) };
    let surface_capabilities = unsafe {
        surface_loader.get_physical_device_surface_capabilities(physical_device, surface)?
    };
    let surface_present_modes = unsafe {
        surface_loader.get_physical_device_surface_present_modes(physical_device, surface)?
    };
    let surface_formats =
        unsafe { surface_loader.get_physical_device_surface_formats(physical_device, surface)? };
    let queuefamilies = [qfamindices.0];
    let swapchain_create_info = vk::SwapchainCreateInfoKHR::builder()
        .surface(surface)
        .min_image_count(
            3.max(surface_capabilities.min_image_count)
                .min(surface_capabilities.max_image_count),
        )
        .image_format(surface_formats.first().unwrap().format)
        .image_color_space(surface_formats.first().unwrap().color_space)
        .image_extent(surface_capabilities.current_extent)
        .image_array_layers(1)
        .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
        .image_sharing_mode(vk::SharingMode::EXCLUSIVE)
        .queue_family_indices(&queuefamilies)
        .pre_transform(surface_capabilities.current_transform)
        .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
        .present_mode(vk::PresentModeKHR::FIFO);
    let swapchain_loader = ash::extensions::khr::Swapchain::new(&instance, &logical_device);
    let swapchain = unsafe { swapchain_loader.create_swapchain(&swapchain_create_info, None)? };
    let swapchain_images = unsafe { swapchain_loader.get_swapchain_images(swapchain)? };
    let mut swapchain_imageviews = Vec::with_capacity(swapchain_images.len());
    for image in &swapchain_images {
        let subresource_range = vk::ImageSubresourceRange::builder()
            .aspect_mask(vk::ImageAspectFlags::COLOR)
            .base_mip_level(0)
            .level_count(1)
            .base_array_layer(0)
            .layer_count(1);
        let imageview_create_info = vk::ImageViewCreateInfo::builder()
            .image(*image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::B8G8R8A8_UNORM)
            .subresource_range(*subresource_range);
        let imageview = unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;
        swapchain_imageviews.push(imageview);
    }

    unsafe {
        for iv in &swapchain_imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
        swapchain_loader.destroy_swapchain(swapchain, None);
        logical_device.destroy_device(None);
        surface_loader.destroy_surface(surface, None);
        debug_utils.destroy_debug_utils_messenger(utils_messenger, None);
        instance.destroy_instance(None)
    };
    Ok(())
}

unsafe extern "system" fn vulkan_debug_utils_callback(
    message_severity: vk::DebugUtilsMessageSeverityFlagsEXT,
    message_type: vk::DebugUtilsMessageTypeFlagsEXT,
    p_callback_data: *const vk::DebugUtilsMessengerCallbackDataEXT,
    _p_user_data: *mut std::ffi::c_void,
) -> vk::Bool32 {
    let message = std::ffi::CStr::from_ptr((*p_callback_data).p_message);
    let severity = format!("{:?}", message_severity).to_lowercase();
    let ty = format!("{:?}", message_type).to_lowercase();
    println!("[Debug][{}][{}] {:?}", severity, ty, message);
    vk::FALSE
}
```

[Continue](008_Cleanup.md)
