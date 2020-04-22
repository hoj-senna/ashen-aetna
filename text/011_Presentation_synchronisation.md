## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Presentation, synchronisation

We're still unable to see the point we wanted to draw. 

Why? 

We have written a list of instructions what we want to be done, but we have not yet given
it to anyone to act on it. 

What else has to happen? 

We have to pass this list of instructions to the queue. 

Which of the lists? (We have created several command buffers,
one for each image in the swapchain.) 

The one for the "right" swapchain image. The one whose turn it currently is. 

New order of business: Get the current image from the swapchain ("acquire" it), submit the commandbuffer, and return the image to the swapchain for drawing it ("present"). 

There could be one problem: What if we tried to acquire an image that is still being drawn? What if we showed one on the screen that is still being drawn
on? That doesn't sound too good. We should take care to never work on objects that someone else is also working on at the same time. We should take
care of the *synchronisation* of work of different actors. 

Whose work do we have to synchronise? First of all, between CPU (our application) and GPU (whatever commands we have sent to Vulkan). 
But also between different things happening on the GPU. GPUs do a lot of work in parallel, and it is a 
good rule of thumb to assume that whatever commands we have given will be executed in random order if we don't give specific instructions. (Actually,
in quite some instances, these commands would be guaranteed to be started in the right order; but in which order they'd be completed, is completely
open.) Vulkan expects us to take care of any necessary synchronisation. 

There are different kinds of synchronisation objects, differing in how disruptive they are (synchronisation means someone has to wait before
continuing their work) or what they are specialised for (CPU/GPU? access of Images? etc.)

What we need here in order to avoid problems like "showing before it is ready", is a `Semaphore`. Semaphores make the following kind of instruction
possible: "when you're done with these operations, give a sign" and "before you start with the following, wait until this sign has been given". That
is: They are things with the state "signaled" or "unsignaled" (binary semaphore). (There is another type with an increasing counter, a "timeline
semaphore".) They are meant for synchronisation of different steps on the GPU.

Let us introduce some into our code. I'll put them with the images and framebuffers to the other swapchain stuff: 
```rust
struct SwapchainDongXi {
    swapchain_loader: ash::extensions::khr::Swapchain,
    swapchain: vk::SwapchainKHR,
    images: Vec<vk::Image>,
    imageviews: Vec<vk::ImageView>,
    framebuffers: Vec<vk::Framebuffer>,
    surface_format: vk::SurfaceFormatKHR,
    extent: vk::Extent2D,
    image_available: Vec<vk::Semaphore>,
    rendering_finished: Vec<vk::Semaphore>,
    amount_of_images: u32,
    current_image: usize,
}
```
(And I have sneakily also inserted a separate member `amount`, because I don't always want to call `.len()` on one of the
images/imageviews/framebuffers, and I've had to do that somewhere before. And another field to keep track of which image is the current one to be
drawing on.) 

For creation, we need a `SemaphoreCreateInfo` — but there's actually nothing to fill in. The interesting part of a semaphore is not some information
given on creation, but just the question whether it is signalled or not. — And when it should be signalled, but that is something we do not have to
say so far in advance.

As a result, the code is not that interesting. I'll note that also cleanup is required and include the whole code for the methods of`SwapchainDongXi` 
```rust
impl SwapchainDongXi {
    fn init(
        instance: &ash::Instance,
        physical_device: vk::PhysicalDevice,
        logical_device: &ash::Device,
        surfaces: &SurfaceDongXi,
        queue_families: &QueueFamilies,
        queues: &Queues,
    ) -> Result<SwapchainDongXi, vk::Result> {
        let surface_capabilities = surfaces.get_capabilities(physical_device)?;
        let extent = surface_capabilities.current_extent;
        let surface_present_modes = surfaces.get_present_modes(physical_device)?;
        let surface_format = *surfaces.get_formats(physical_device)?.first().unwrap();
        let queuefamilies = [queue_families.graphics_q_index.unwrap()];
        let swapchain_create_info = vk::SwapchainCreateInfoKHR::builder()
            .surface(surfaces.surface)
            .min_image_count(
                3.max(surface_capabilities.min_image_count)
                    .min(surface_capabilities.max_image_count),
            )
            .image_format(surface_format.format)
            .image_color_space(surface_format.color_space)
            .image_extent(extent)
            .image_array_layers(1)
            .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
            .image_sharing_mode(vk::SharingMode::EXCLUSIVE)
            .queue_family_indices(&queuefamilies)
            .pre_transform(surface_capabilities.current_transform)
            .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
            .present_mode(vk::PresentModeKHR::FIFO);
        let swapchain_loader = ash::extensions::khr::Swapchain::new(instance, logical_device);
        let swapchain = unsafe { swapchain_loader.create_swapchain(&swapchain_create_info, None)? };
        let swapchain_images = unsafe { swapchain_loader.get_swapchain_images(swapchain)? };
        let amount_of_images = swapchain_images.len() as u32;
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
            let imageview =
                unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;
            swapchain_imageviews.push(imageview);
        }
        let mut image_available = vec![];
        let mut rendering_finished = vec![];
        let semaphoreinfo = vk::SemaphoreCreateInfo::builder();
        for _ in 0..amount_of_images {
            let semaphore_available =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            let semaphore_finished =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            image_available.push(semaphore_available);
            rendering_finished.push(semaphore_finished);
        }
        Ok(SwapchainDongXi {
            swapchain_loader,
            swapchain,
            images: swapchain_images,
            imageviews: swapchain_imageviews,
            framebuffers: vec![],
            surface_format,
            extent,
            amount_of_images,
            current_image:0,
            image_available,
            rendering_finished,
        })
    }
    fn create_framebuffers(
        &mut self,
        logical_device: &ash::Device,
        renderpass: vk::RenderPass,
    ) -> Result<(), vk::Result> {
        for iv in &self.imageviews {
            let iview = [*iv];
            let framebuffer_info = vk::FramebufferCreateInfo::builder()
                .render_pass(renderpass)
                .attachments(&iview)
                .width(self.extent.width)
                .height(self.extent.height)
                .layers(1);
            let fb = unsafe { logical_device.create_framebuffer(&framebuffer_info, None) }?;
            self.framebuffers.push(fb);
        }
        Ok(())
    }
    unsafe fn cleanup(&mut self, logical_device: &ash::Device) {
        for semaphore in &self.image_available {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for semaphore in &self.rendering_finished {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for fb in &self.framebuffers {
            logical_device.destroy_framebuffer(*fb, None);
        }
        for iv in &self.imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
        self.swapchain_loader
            .destroy_swapchain(self.swapchain, None)
    }
}
```

When we acquire the image from the swapchain, it is possible that its presentation on the screen is not yet over. The
[spec](https://www.khronos.org/registry/vulkan/specs/1.0-wsi_extensions/html/vkspec.html#_wsi_swapchain) explicitly gives us some 
homework: "After acquiring a presentable image and before modifying it, the application must use a synchronization primitive to ensure that the
presentation engine has finished reading from the image": 

More synchronisation is needed. 

Another semaphore? No, this time it's synchronisation between GPU and CPU and the "semaphores" for that have a different name: 
They are "fences", and instead of merely including an instruction "before doing the following, wait until this semaphore is signalled", 
we can explicitly ask for the status of a fence in our application code.   
```rust
struct SwapchainDongXi {
    swapchain_loader: ash::extensions::khr::Swapchain,
    swapchain: vk::SwapchainKHR,
    images: Vec<vk::Image>,
    imageviews: Vec<vk::ImageView>,
    framebuffers: Vec<vk::Framebuffer>,
    surface_format: vk::SurfaceFormatKHR,
    extent: vk::Extent2D,
    image_available: Vec<vk::Semaphore>,
    rendering_finished: Vec<vk::Semaphore>,
    may_begin_drawing: Vec<vk::Fence>,
    amount_of_images: u32,
    current_image: usize,
}
```
Upon creation, a fence by default is in the unsignaled state. That's not appropriate here; at the beginning there is no presentation to wait for. So
let's create a signaled fence: 
```rust
        let fenceinfo = vk::FenceCreateInfo::builder().flags(vk::FenceCreateFlags::SIGNALED);
```
or rather: several of them: 
```rust
        let mut image_available = vec![];
        let mut rendering_finished = vec![];
        let mut may_begin_drawing = vec![];
        let semaphoreinfo = vk::SemaphoreCreateInfo::builder();
        let fenceinfo = vk::FenceCreateInfo::builder().flags(vk::FenceCreateFlags::SIGNALED);
        for _ in 0..amount_of_images {
            let semaphore_available =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            let semaphore_finished =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            image_available.push(semaphore_available);
            rendering_finished.push(semaphore_finished);
            let fence = unsafe { logical_device.create_fence(&fenceinfo, None) }?;
            may_begin_drawing.push(fence);
        }
```
And let's not forget about the cleanup: 
```rust
    unsafe fn cleanup(&mut self, logical_device: &ash::Device) {
        for fence in &self.may_begin_drawing {
            logical_device.destroy_fence(*fence, None);
        }
        for semaphore in &self.image_available {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for semaphore in &self.rendering_finished {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for fb in &self.framebuffers {
            logical_device.destroy_framebuffer(*fb, None);
        }
        for iv in &self.imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
        self.swapchain_loader
            .destroy_swapchain(self.swapchain, None)
    }
```
That's the synchronisation taken care of. 



Now, let's head over to this part: 
```rust
        Event::RedrawRequested(_) => {
            //render here (later)
        }
```
and add something in there. After so many preparations, we should finally submit some other commands than "prepare something for later" or "destroy"
to Vulkan.
Firstly (or, maybe, lastly), we count whose turn it is: Do we wait for semaphores and fences no 0, no 1 or no 2?
```rust
            aetna.swapchain.current_image =
                (aetna.swapchain.current_image + 1) % aetna.swapchain.amount_of_images as usize;
```
We acquire the next image. This command gives us the index of image/framebuffer to use in the corresponding vector (and also answers the question "is
something wrong with the swapchain (like being invalidated from a change of window size)?" — but we don't care for now). 
```rust
            let (image_index, _) = unsafe {
                aetna
                    .swapchain
                    .swapchain_loader
                    .acquire_next_image(
                        aetna.swapchain.swapchain,
                        std::u64::MAX,
                        aetna.swapchain.image_available[aetna.swapchain.current_image],
                        vk::Fence::null(),
                    )
                    .expect("image acquisition trouble")
            };
```
The first argument used is the swapchain; the second a waiting time: If no image can be found, how long should the function wait? In nanoseconds; we
take "as many as necessary" or the closest approximation. The last two arguments are a semaphore and a fence that should be signaled when execution of
"acquire image" is completed. (We cannot use "null" for both.) 

Next we wait for the fence (that's what we have created it for): 
```rust
                aetna
                    .device
                    .wait_for_fences(
                        &[aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image]],
                        true,
                        std::u64::MAX,
                    )
                    .expect("fence-waiting");
```
This command again expects a maximal waiting time, and apart from that, an array of fences. How long should we wait? Until all fences are signaled? Or
just until at least one is signaled? This is the boolean argument: "wait for all?" — as long as it's just one fence, this should not matter.
Having waited for the fence, we should return it into its unsignaled state: 
```rust
                aetna
                    .device
                    .reset_fences(&[
                        aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image]
                    ])
                    .expect("resetting fences");
```
Next, we submit the commandbuffer. Obviously (hey, it's Vulkan!), this needs some `SubmitInfo`: 
```rust
            let semaphores_available = [aetna.swapchain.image_available[aetna.swapchain.current_image]];
            let waiting_stages = [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
            let semaphores_finished = [aetna.swapchain.rendering_finished[aetna.swapchain.current_image]];
            let commandbuffers = [aetna.commandbuffers[image_index as usize]];
            let submit_info = [vk::SubmitInfo::builder()
                .wait_semaphores(&semaphores_available)
                .wait_dst_stage_mask(&waiting_stages)
                .command_buffers(&commandbuffers)
                .signal_semaphores(&semaphores_finished)
                .build()];
```
Which semaphores to wait for? At (before) which stages of the pipeline should we wait? Which command buffer is to be submitted? And: which semaphores
to signal after execution? 

Then we submit (asking for the fence to be signaled afterwards): 
```rust
            unsafe {
                aetna
                    .device
                    .queue_submit(
                        aetna.queues.graphics_queue,
                        &submit_info,
                        aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image],
                    )
                    .expect("queue submission");
            };
```
Still nothing to be seen? But it is drawn already, we only have to present it on the screen (specifying for which semaphores to wait, which swapchain
and which images to use): 
```rust
            let swapchains = [aetna.swapchain.swapchain];
            let indices = [image_index];
            let present_info = vk::PresentInfoKHR::builder()
                .wait_semaphores(&semaphores_finished)
                .swapchains(&swapchains)
                .image_indices(&indices);
            unsafe {
                aetna
                    .swapchain
                    .swapchain_loader
                    .queue_present(aetna.queues.graphics_queue, &present_info)
                    .expect("queue presentation");
            };
```
And, I proudly present to you: A dark screen with a red dot in the middle. 
A quick check of the messages from the validation layers, however, reveals that not all is well: 

```
[Debug][error][validation] "Attempt to destroy command pool with command buffer (0x565467d45650) which is in use. The Vulkan spec states: All
VkCommandBuffer objects allocated from commandPool must not be in the pending state.
(https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyCommandPool-commandPool-00041)"
```
and many similar error messages appear. Fortunately, they give a good hint what is happening: We abruptly try to destroy the objects while the GPU is
still working with them. Some more synchronisation is needed, it seems. This time in our `.drop()` function. 
```rust
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
```
Waiting until the device is idle is a much less fine-grained synchronisation tool than semaphores, fences or other synchronisation primitives we will
come across later, and should be thought of as slow and something to avoid. This, immediately before shutting down the program, is, however, the right
situation to apply it. 


This is a reasonable place to begin if you do not want to *start* with the Vulkan setup. Let me, therefore, include more than just our main.rs. 
The dependencies section in Cargo.toml: 
```rust
[dependencies]
ash = "0.30.0"
winit = "0.22.0"
vk-shader-macros = "0.2.2"
```
vertex shader in shaders/shader.vert: 
```glsl
	#version 450

	void main() {
	    gl_PointSize=10.0;
	    gl_Position = vec4(0.0,0.0,0.0,1.0);
	}
```
fragment shader in shaders/shader.frag: 
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	void main(){
		theColour= vec4(1.0,0.0,0.0,1.0);
	}
```

```rust
use ash::version::DeviceV1_0;
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;
use ash::vk;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;
    let mut aetna = Aetna::init(window)?;
    use winit::event::{Event, WindowEvent};
    eventloop.run(move |event, _, controlflow| match event {
        Event::WindowEvent {
            event: WindowEvent::CloseRequested,
            ..
        } => {
            *controlflow = winit::event_loop::ControlFlow::Exit;
        }
        Event::MainEventsCleared => {
            // doing the work here (later)
            aetna.window.request_redraw();
        }
        Event::RedrawRequested(_) => {
            let (image_index, _) = unsafe {
                aetna
                    .swapchain
                    .swapchain_loader
                    .acquire_next_image(
                        aetna.swapchain.swapchain,
                        std::u64::MAX,
                        aetna.swapchain.image_available[aetna.swapchain.current_image],
                        vk::Fence::null(),
                    )
                    .expect("image acquisition trouble")
            };
            unsafe {
                aetna
                    .device
                    .wait_for_fences(
                        &[aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image]],
                        true,
                        std::u64::MAX,
                    )
                    .expect("fence-waiting");
                aetna
                    .device
                    .reset_fences(&[
                        aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image]
                    ])
                    .expect("resetting fences");
            }
            let semaphores_available =
                [aetna.swapchain.image_available[aetna.swapchain.current_image]];
            let waiting_stages = [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
            let semaphores_finished =
                [aetna.swapchain.rendering_finished[aetna.swapchain.current_image]];
            let commandbuffers = [aetna.commandbuffers[image_index as usize]];
            let submit_info = [vk::SubmitInfo::builder()
                .wait_semaphores(&semaphores_available)
                .wait_dst_stage_mask(&waiting_stages)
                .command_buffers(&commandbuffers)
                .signal_semaphores(&semaphores_finished)
                .build()];
            unsafe {
                aetna
                    .device
                    .queue_submit(
                        aetna.queues.graphics_queue,
                        &submit_info,
                        aetna.swapchain.may_begin_drawing[aetna.swapchain.current_image],
                    )
                    .expect("queue submission");
            };
            let swapchains = [aetna.swapchain.swapchain];
            let indices = [image_index];
            let present_info = vk::PresentInfoKHR::builder()
                .wait_semaphores(&semaphores_finished)
                .swapchains(&swapchains)
                .image_indices(&indices);
            unsafe {
                aetna
                    .swapchain
                    .swapchain_loader
                    .queue_present(aetna.queues.graphics_queue, &present_info)
                    .expect("queue presentation");
            };
            aetna.swapchain.current_image =
                (aetna.swapchain.current_image + 1) % aetna.swapchain.amount_of_images as usize;
        }
        _ => {}
    });
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

fn init_instance(
    entry: &ash::Entry,
    layer_names: &[&str],
) -> Result<ash::Instance, ash::InstanceError> {
    let enginename = std::ffi::CString::new("UnknownGameEngine").unwrap();
    let appname = std::ffi::CString::new("The Black Window").unwrap();
    let app_info = vk::ApplicationInfo::builder()
        .application_name(&appname)
        .application_version(vk::make_version(0, 0, 1))
        .engine_name(&enginename)
        .engine_version(vk::make_version(0, 42, 0))
        .api_version(vk::make_version(1, 0, 106));
    let layer_names_c: Vec<std::ffi::CString> = layer_names
        .iter()
        .map(|&ln| std::ffi::CString::new(ln).unwrap())
        .collect();
    let layer_name_pointers: Vec<*const i8> = layer_names_c
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
    unsafe { entry.create_instance(&instance_create_info, None) }
}

struct DebugDongXi {
    loader: ash::extensions::ext::DebugUtils,
    messenger: vk::DebugUtilsMessengerEXT,
}
impl DebugDongXi {
    fn init(entry: &ash::Entry, instance: &ash::Instance) -> Result<DebugDongXi, vk::Result> {
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

        let loader = ash::extensions::ext::DebugUtils::new(entry, instance);
        let messenger = unsafe { loader.create_debug_utils_messenger(&debugcreateinfo, None)? };

        Ok(DebugDongXi { loader, messenger })
    }
}

impl Drop for DebugDongXi {
    fn drop(&mut self) {
        unsafe {
            self.loader
                .destroy_debug_utils_messenger(self.messenger, None)
        };
    }
}

struct SurfaceDongXi {
    xlib_surface_loader: ash::extensions::khr::XlibSurface,
    surface: vk::SurfaceKHR,
    surface_loader: ash::extensions::khr::Surface,
}

impl SurfaceDongXi {
    fn init(
        window: &winit::window::Window,
        entry: &ash::Entry,
        instance: &ash::Instance,
    ) -> Result<SurfaceDongXi, vk::Result> {
        use winit::platform::unix::WindowExtUnix;
        let x11_display = window.xlib_display().unwrap();
        let x11_window = window.xlib_window().unwrap();
        let x11_create_info = vk::XlibSurfaceCreateInfoKHR::builder()
            .window(x11_window)
            .dpy(x11_display as *mut vk::Display);
        let xlib_surface_loader = ash::extensions::khr::XlibSurface::new(entry, instance);
        let surface = unsafe { xlib_surface_loader.create_xlib_surface(&x11_create_info, None) }?;
        let surface_loader = ash::extensions::khr::Surface::new(entry, instance);
        Ok(SurfaceDongXi {
            xlib_surface_loader,
            surface,
            surface_loader,
        })
    }
    fn get_capabilities(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<vk::SurfaceCapabilitiesKHR, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_capabilities(physical_device, self.surface)
        }
    }
    fn get_present_modes(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Vec<vk::PresentModeKHR>, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_present_modes(physical_device, self.surface)
        }
    }
    fn get_formats(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Vec<vk::SurfaceFormatKHR>, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_formats(physical_device, self.surface)
        }
    }
    fn get_physical_device_surface_support(
        &self,
        physical_device: vk::PhysicalDevice,
        queuefamilyindex: usize,
    ) -> Result<bool, vk::Result> {
        unsafe {
            self.surface_loader.get_physical_device_surface_support(
                physical_device,
                queuefamilyindex as u32,
                self.surface,
            )
        }
    }
}

impl Drop for SurfaceDongXi {
    fn drop(&mut self) {
        unsafe {
            self.surface_loader.destroy_surface(self.surface, None);
        }
    }
}

fn init_physical_device_and_properties(
    instance: &ash::Instance,
) -> Result<(vk::PhysicalDevice, vk::PhysicalDeviceProperties), vk::Result> {
    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    let mut chosen = None;
    for p in phys_devs {
        let properties = unsafe { instance.get_physical_device_properties(p) };
        if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
            chosen = Some((p, properties));
        }
    }
    Ok(chosen.unwrap())
}

struct QueueFamilies {
    graphics_q_index: Option<u32>,
    transfer_q_index: Option<u32>,
}
impl QueueFamilies {
    fn init(
        instance: &ash::Instance,
        physical_device: vk::PhysicalDevice,
        surfaces: &SurfaceDongXi,
    ) -> Result<QueueFamilies, vk::Result> {
        let queuefamilyproperties =
            unsafe { instance.get_physical_device_queue_family_properties(physical_device) };
        let mut found_graphics_q_index = None;
        let mut found_transfer_q_index = None;
        for (index, qfam) in queuefamilyproperties.iter().enumerate() {
            if qfam.queue_count > 0
                && qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
                && surfaces.get_physical_device_surface_support(physical_device, index)?
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
        Ok(QueueFamilies {
            graphics_q_index: found_graphics_q_index,
            transfer_q_index: found_transfer_q_index,
        })
    }
}

struct Queues {
    graphics_queue: vk::Queue,
    transfer_queue: vk::Queue,
}

fn init_device_and_queues(
    instance: &ash::Instance,
    physical_device: vk::PhysicalDevice,
    queue_families: &QueueFamilies,
    layer_names: &[&str],
) -> Result<(ash::Device, Queues), vk::Result> {
    let layer_names_c: Vec<std::ffi::CString> = layer_names
        .iter()
        .map(|&ln| std::ffi::CString::new(ln).unwrap())
        .collect();
    let layer_name_pointers: Vec<*const i8> = layer_names_c
        .iter()
        .map(|layer_name| layer_name.as_ptr())
        .collect();

    let priorities = [1.0f32];
    let queue_infos = [
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(queue_families.graphics_q_index.unwrap())
            .queue_priorities(&priorities)
            .build(),
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(queue_families.transfer_q_index.unwrap())
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
    let graphics_queue =
        unsafe { logical_device.get_device_queue(queue_families.graphics_q_index.unwrap(), 0) };
    let transfer_queue =
        unsafe { logical_device.get_device_queue(queue_families.transfer_q_index.unwrap(), 0) };
    Ok((
        logical_device,
        Queues {
            graphics_queue,
            transfer_queue,
        },
    ))
}

struct SwapchainDongXi {
    swapchain_loader: ash::extensions::khr::Swapchain,
    swapchain: vk::SwapchainKHR,
    images: Vec<vk::Image>,
    imageviews: Vec<vk::ImageView>,
    framebuffers: Vec<vk::Framebuffer>,
    surface_format: vk::SurfaceFormatKHR,
    extent: vk::Extent2D,
    image_available: Vec<vk::Semaphore>,
    rendering_finished: Vec<vk::Semaphore>,
    may_begin_drawing: Vec<vk::Fence>,
    amount_of_images: u32,
    current_image: usize,
}

impl SwapchainDongXi {
    fn init(
        instance: &ash::Instance,
        physical_device: vk::PhysicalDevice,
        logical_device: &ash::Device,
        surfaces: &SurfaceDongXi,
        queue_families: &QueueFamilies,
        queues: &Queues,
    ) -> Result<SwapchainDongXi, vk::Result> {
        let surface_capabilities = surfaces.get_capabilities(physical_device)?;
        let extent = surface_capabilities.current_extent;
        let surface_present_modes = surfaces.get_present_modes(physical_device)?;
        let surface_format = *surfaces.get_formats(physical_device)?.first().unwrap();
        let queuefamilies = [queue_families.graphics_q_index.unwrap()];
        let swapchain_create_info = vk::SwapchainCreateInfoKHR::builder()
            .surface(surfaces.surface)
            .min_image_count(
                3.max(surface_capabilities.min_image_count)
                    .min(surface_capabilities.max_image_count),
            )
            .image_format(surface_format.format)
            .image_color_space(surface_format.color_space)
            .image_extent(extent)
            .image_array_layers(1)
            .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
            .image_sharing_mode(vk::SharingMode::EXCLUSIVE)
            .queue_family_indices(&queuefamilies)
            .pre_transform(surface_capabilities.current_transform)
            .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
            .present_mode(vk::PresentModeKHR::FIFO);
        let swapchain_loader = ash::extensions::khr::Swapchain::new(instance, logical_device);
        let swapchain = unsafe { swapchain_loader.create_swapchain(&swapchain_create_info, None)? };
        let swapchain_images = unsafe { swapchain_loader.get_swapchain_images(swapchain)? };
        let amount_of_images = swapchain_images.len() as u32;
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
            let imageview =
                unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;
            swapchain_imageviews.push(imageview);
        }
        let mut image_available = vec![];
        let mut rendering_finished = vec![];
        let mut may_begin_drawing = vec![];
        let semaphoreinfo = vk::SemaphoreCreateInfo::builder();
        let fenceinfo = vk::FenceCreateInfo::builder().flags(vk::FenceCreateFlags::SIGNALED);
        for _ in 0..amount_of_images {
            let semaphore_available =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            let semaphore_finished =
                unsafe { logical_device.create_semaphore(&semaphoreinfo, None) }?;
            image_available.push(semaphore_available);
            rendering_finished.push(semaphore_finished);
            let fence = unsafe { logical_device.create_fence(&fenceinfo, None) }?;
            may_begin_drawing.push(fence);
        }
        Ok(SwapchainDongXi {
            swapchain_loader,
            swapchain,
            images: swapchain_images,
            imageviews: swapchain_imageviews,
            framebuffers: vec![],
            surface_format,
            extent,
            amount_of_images,
            current_image: 0,
            image_available,
            rendering_finished,
            may_begin_drawing,
        })
    }
    fn create_framebuffers(
        &mut self,
        logical_device: &ash::Device,
        renderpass: vk::RenderPass,
    ) -> Result<(), vk::Result> {
        for iv in &self.imageviews {
            let iview = [*iv];
            let framebuffer_info = vk::FramebufferCreateInfo::builder()
                .render_pass(renderpass)
                .attachments(&iview)
                .width(self.extent.width)
                .height(self.extent.height)
                .layers(1);
            let fb = unsafe { logical_device.create_framebuffer(&framebuffer_info, None) }?;
            self.framebuffers.push(fb);
        }
        Ok(())
    }
    unsafe fn cleanup(&mut self, logical_device: &ash::Device) {
        for fence in &self.may_begin_drawing {
            logical_device.destroy_fence(*fence, None);
        }
        for semaphore in &self.image_available {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for semaphore in &self.rendering_finished {
            logical_device.destroy_semaphore(*semaphore, None);
        }
        for fb in &self.framebuffers {
            logical_device.destroy_framebuffer(*fb, None);
        }
        for iv in &self.imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
        self.swapchain_loader
            .destroy_swapchain(self.swapchain, None)
    }
}

fn init_renderpass(
    logical_device: &ash::Device,
    physical_device: vk::PhysicalDevice,
    format: vk::Format,
) -> Result<vk::RenderPass, vk::Result> {
    let attachments = [vk::AttachmentDescription::builder()
        .format(format)
        .load_op(vk::AttachmentLoadOp::CLEAR)
        .store_op(vk::AttachmentStoreOp::STORE)
        .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
        .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
        .initial_layout(vk::ImageLayout::UNDEFINED)
        .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
        .samples(vk::SampleCountFlags::TYPE_1)
        .build()];
    let color_attachment_references = [vk::AttachmentReference {
        attachment: 0,
        layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
    }];
    let subpasses = [vk::SubpassDescription::builder()
        .color_attachments(&color_attachment_references)
        .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
        .build()];
    let subpass_dependencies = [vk::SubpassDependency::builder()
        .src_subpass(vk::SUBPASS_EXTERNAL)
        .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_subpass(0)
        .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_access_mask(
            vk::AccessFlags::COLOR_ATTACHMENT_READ | vk::AccessFlags::COLOR_ATTACHMENT_WRITE,
        )
        .build()];
    let renderpass_info = vk::RenderPassCreateInfo::builder()
        .attachments(&attachments)
        .subpasses(&subpasses)
        .dependencies(&subpass_dependencies);
    let renderpass = unsafe { logical_device.create_render_pass(&renderpass_info, None)? };
    Ok(renderpass)
}

struct Pipeline {
    pipeline: vk::Pipeline,
    layout: vk::PipelineLayout,
}

impl Pipeline {
    fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            logical_device.destroy_pipeline(self.pipeline, None);
            logical_device.destroy_pipeline_layout(self.layout, None);
        }
    }

    fn init(
        logical_device: &ash::Device,
        swapchain: &SwapchainDongXi,
        renderpass: &vk::RenderPass,
    ) -> Result<Pipeline, vk::Result> {
        let vertexshader_createinfo = vk::ShaderModuleCreateInfo::builder().code(
            vk_shader_macros::include_glsl!("./shaders/shader.vert", kind: vert),
        );
        let vertexshader_module =
            unsafe { logical_device.create_shader_module(&vertexshader_createinfo, None)? };
        let fragmentshader_createinfo = vk::ShaderModuleCreateInfo::builder()
            .code(vk_shader_macros::include_glsl!("./shaders/shader.frag"));
        let fragmentshader_module =
            unsafe { logical_device.create_shader_module(&fragmentshader_createinfo, None)? };
        let mainfunctionname = std::ffi::CString::new("main").unwrap();
        let vertexshader_stage = vk::PipelineShaderStageCreateInfo::builder()
            .stage(vk::ShaderStageFlags::VERTEX)
            .module(vertexshader_module)
            .name(&mainfunctionname);
        let fragmentshader_stage = vk::PipelineShaderStageCreateInfo::builder()
            .stage(vk::ShaderStageFlags::FRAGMENT)
            .module(fragmentshader_module)
            .name(&mainfunctionname);
        let shader_stages = vec![vertexshader_stage.build(), fragmentshader_stage.build()];
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder();
        let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::POINT_LIST);
        let viewports = [vk::Viewport {
            x: 0.,
            y: 0.,
            width: swapchain.extent.width as f32,
            height: swapchain.extent.height as f32,
            min_depth: 0.,
            max_depth: 1.,
        }];
        let scissors = [vk::Rect2D {
            offset: vk::Offset2D { x: 0, y: 0 },
            extent: swapchain.extent,
        }];

        let viewport_info = vk::PipelineViewportStateCreateInfo::builder()
            .viewports(&viewports)
            .scissors(&scissors);
        let rasterizer_info = vk::PipelineRasterizationStateCreateInfo::builder()
            .line_width(1.0)
            .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
            .cull_mode(vk::CullModeFlags::NONE)
            .polygon_mode(vk::PolygonMode::FILL);
        let multisampler_info = vk::PipelineMultisampleStateCreateInfo::builder()
            .rasterization_samples(vk::SampleCountFlags::TYPE_1);
        let colourblend_attachments = [vk::PipelineColorBlendAttachmentState::builder()
            .blend_enable(true)
            .src_color_blend_factor(vk::BlendFactor::SRC_ALPHA)
            .dst_color_blend_factor(vk::BlendFactor::ONE_MINUS_SRC_ALPHA)
            .color_blend_op(vk::BlendOp::ADD)
            .src_alpha_blend_factor(vk::BlendFactor::SRC_ALPHA)
            .dst_alpha_blend_factor(vk::BlendFactor::ONE_MINUS_SRC_ALPHA)
            .alpha_blend_op(vk::BlendOp::ADD)
            .color_write_mask(
                vk::ColorComponentFlags::R
                    | vk::ColorComponentFlags::G
                    | vk::ColorComponentFlags::B
                    | vk::ColorComponentFlags::A,
            )
            .build()];
        let colourblend_info =
            vk::PipelineColorBlendStateCreateInfo::builder().attachments(&colourblend_attachments);
        let pipelinelayout_info = vk::PipelineLayoutCreateInfo::builder();
        let pipelinelayout =
            unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
        let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&shader_stages)
            .vertex_input_state(&vertex_input_info)
            .input_assembly_state(&input_assembly_info)
            .viewport_state(&viewport_info)
            .rasterization_state(&rasterizer_info)
            .multisample_state(&multisampler_info)
            .color_blend_state(&colourblend_info)
            .layout(pipelinelayout)
            .render_pass(*renderpass)
            .subpass(0);
        let graphicspipeline = unsafe {
            logical_device
                .create_graphics_pipelines(
                    vk::PipelineCache::null(),
                    &[pipeline_info.build()],
                    None,
                )
                .expect("A problem with the pipeline creation")
        }[0];
        unsafe {
            logical_device.destroy_shader_module(fragmentshader_module, None);
            logical_device.destroy_shader_module(vertexshader_module, None);
        }
        Ok(Pipeline {
            pipeline: graphicspipeline,
            layout: pipelinelayout,
        })
    }
}

struct Pools {
    commandpool_graphics: vk::CommandPool,
    commandpool_transfer: vk::CommandPool,
}

impl Pools {
    fn init(
        logical_device: &ash::Device,
        queue_families: &QueueFamilies,
    ) -> Result<Pools, vk::Result> {
        let graphics_commandpool_info = vk::CommandPoolCreateInfo::builder()
            .queue_family_index(queue_families.graphics_q_index.unwrap())
            .flags(vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER);
        let commandpool_graphics =
            unsafe { logical_device.create_command_pool(&graphics_commandpool_info, None) }?;
        let transfer_commandpool_info = vk::CommandPoolCreateInfo::builder()
            .queue_family_index(queue_families.transfer_q_index.unwrap())
            .flags(vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER);
        let commandpool_transfer =
            unsafe { logical_device.create_command_pool(&transfer_commandpool_info, None) }?;

        Ok(Pools {
            commandpool_graphics,
            commandpool_transfer,
        })
    }
    fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            logical_device.destroy_command_pool(self.commandpool_graphics, None);
            logical_device.destroy_command_pool(self.commandpool_transfer, None);
        }
    }
}

fn create_commandbuffers(
    logical_device: &ash::Device,
    pools: &Pools,
    amount: u32,
) -> Result<Vec<vk::CommandBuffer>, vk::Result> {
    let commandbuf_allocate_info = vk::CommandBufferAllocateInfo::builder()
        .command_pool(pools.commandpool_graphics)
        .command_buffer_count(amount);
    unsafe { logical_device.allocate_command_buffers(&commandbuf_allocate_info) }
}

fn fill_commandbuffers(
    commandbuffers: &[vk::CommandBuffer],
    logical_device: &ash::Device,
    renderpass: &vk::RenderPass,
    swapchain: &SwapchainDongXi,
    pipeline: &Pipeline,
) -> Result<(), vk::Result> {
    for (i, &commandbuffer) in commandbuffers.iter().enumerate() {
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            logical_device.begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [vk::ClearValue {
            color: vk::ClearColorValue {
                float32: [0.0, 0.0, 0.08, 1.0],
            },
        }];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(*renderpass)
            .framebuffer(swapchain.framebuffers[i])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: swapchain.extent,
            })
            .clear_values(&clearvalues);
        unsafe {
            logical_device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            logical_device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                pipeline.pipeline,
            );
            logical_device.cmd_draw(commandbuffer, 1, 1, 0, 0);
            logical_device.cmd_end_render_pass(commandbuffer);
            logical_device.end_command_buffer(commandbuffer)?;
        }
    }
    Ok(())
}

struct Aetna {
    window: winit::window::Window,
    entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    physical_device: vk::PhysicalDevice,
    physical_device_properties: vk::PhysicalDeviceProperties,
    queue_families: QueueFamilies,
    queues: Queues,
    device: ash::Device,
    swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    commandbuffers: Vec<vk::CommandBuffer>,
}

impl Aetna {
    fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;
        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &queues,
        )?;
        let renderpass = init_renderpass(
            &logical_device,
            physical_device,
            swapchain.surface_format.format,
        )?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;
        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;
        fill_commandbuffers(
            &commandbuffers,
            &logical_device,
            &renderpass,
            &swapchain,
            &pipeline,
        )?;

        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
        })
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device);
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
```

855 lines (main.rs + shaders), and a single dot on the screen. But, as mentioned in the beginning, the way from nothing to the first dot (okay, there
I said triangle) is the longest and most difficult part of the Vulkan journey. (To be fair, that's a somewhat bold claim to make for someone whose
Vulkan journey exceeds this initial walk by a few small steps only. But the sentiment seems to be shared by more experienced programmers.) We have
touched on so many topics already that we have done much more than "just drawing a dot" and have at least some idea of what all is happening behind
the scenes. Before we go on to turn this "dot on screen" program into something at least slightly more exciting, let us have a [look](012_Looking_back.md) 
back.
