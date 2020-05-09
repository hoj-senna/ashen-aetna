## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Cleanup 
It is not my favourite part, but I think it is time: `main.rs` is growing rather long, with some parts of the code that I don't touch in most of the
new chapters. There are some hints by Clippy that I've been ignoring for some time now. I should clean up.

(The structure of what emerges is not well thought out, it's more a collection of spur-of-the-moment decisions. It's possible I'll revise it later.
But I feel I can't ignore the largeness of the file much longer.)

For reference: This is the (one and only) Rust file: 
```rust
use ash::version::DeviceV1_0;
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;
use ash::vk;
use nalgebra as na;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;
    let mut aetna = Aetna::init(window)?;
    let mut cube = Model::cube();
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.0, 0.1))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [0.2, 0.4, 1.0],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.05, 0.05, 0.0))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [1.0, 1.0, 0.2],
    });
    for i in 0..10 {
        for j in 0..10 {
            cube.insert_visibly(InstanceData {
                modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(
                    i as f32 * 0.2 - 1.0,
                    j as f32 * 0.2 - 1.0,
                    0.5,
                )) * na::Matrix4::new_scaling(0.03))
                .into(),
                colour: [1.0, i as f32 * 0.07, j as f32 * 0.07],
            });
            cube.insert_visibly(InstanceData {
                modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(
                    i as f32 * 0.2 - 1.0,
                    0.0,
                    j as f32 * 0.2 - 1.0,
                )) * na::Matrix4::new_scaling(0.02))
                .into(),
                colour: [i as f32 * 0.07, j as f32 * 0.07, 1.0],
            });
        }
    }
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::from_scaled_axis(na::Vector3::new(0.0, 0.0, 1.4))
            * na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [0.0, 0.5, 0.0],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.5, 0.0, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.5, 0.01, 0.01)))
        .into(),
        colour: [1.0, 0.5, 0.5],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.01, 0.5, 0.01)))
        .into(),
        colour: [0.5, 1.0, 0.5],
    });

    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.0, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.01, 0.01, 0.5)))
        .into(),
        colour: [0.5, 0.5, 1.0],
    });
    cube.update_vertexbuffer(&aetna.allocator);
    cube.update_indexbuffer(&aetna.allocator);
    cube.update_instancebuffer(&aetna.allocator);
    aetna.models = vec![cube];
    let mut camera = Camera::builder().build();
    use winit::event::{Event, WindowEvent};
    eventloop.run(move |event, _, controlflow| match event {
        Event::WindowEvent {
            event: WindowEvent::CloseRequested,
            ..
        } => {
            *controlflow = winit::event_loop::ControlFlow::Exit;
        }
        Event::WindowEvent {
            event: WindowEvent::KeyboardInput { input, .. },
            ..
        } => {
            if let winit::event::KeyboardInput {
                state: winit::event::ElementState::Pressed,
                virtual_keycode: Some(keycode),
                ..
            } = input
            {
                match keycode {
                    winit::event::VirtualKeyCode::Right => {
                        camera.turn_right(0.1);
                    }
                    winit::event::VirtualKeyCode::Left => {
                        camera.turn_left(0.1);
                    }
                    winit::event::VirtualKeyCode::Up => {
                        camera.move_forward(0.05);
                    }
                    winit::event::VirtualKeyCode::Down => {
                        camera.move_backward(0.05);
                    }
                    winit::event::VirtualKeyCode::PageUp => {
                        camera.turn_up(0.02);
                    }
                    winit::event::VirtualKeyCode::PageDown => {
                        camera.turn_down(0.02);
                    }
                    _ => {}
                }
            }
        }
        Event::MainEventsCleared => {
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
            camera.update_buffer(&aetna.allocator, &mut aetna.uniformbuffer);
            for m in &mut aetna.models {
                m.update_instancebuffer(&aetna.allocator);
            }
            aetna
                .update_commandbuffer(image_index as usize)
                .expect("updating the command buffer");
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
        let debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
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
) -> Result<
    (
        vk::PhysicalDevice,
        vk::PhysicalDeviceProperties,
        vk::PhysicalDeviceFeatures,
    ),
    vk::Result,
> {
    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    let mut chosen = None;
    for p in phys_devs {
        let properties = unsafe { instance.get_physical_device_properties(p) };
        let features = unsafe { instance.get_physical_device_features(p) };
        if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
            chosen = Some((p, properties, features));
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
    let features = vk::PhysicalDeviceFeatures::builder().fill_mode_non_solid(true);
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_features(&features);
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
    depth_image: vk::Image,
    depth_image_allocation: vk_mem::Allocation,
    depth_image_allocation_info: vk_mem::AllocationInfo,
    depth_imageview: vk::ImageView,
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
        allocator: &vk_mem::Allocator,
    ) -> Result<SwapchainDongXi, Box<dyn std::error::Error>> {
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
        let extent3d = vk::Extent3D {
            width: extent.width,
            height: extent.height,
            depth: 1,
        };
        let depth_image_info = vk::ImageCreateInfo::builder()
            .image_type(vk::ImageType::TYPE_2D)
            .format(vk::Format::D32_SFLOAT)
            .extent(extent3d)
            .mip_levels(1)
            .array_layers(1)
            .samples(vk::SampleCountFlags::TYPE_1)
            .tiling(vk::ImageTiling::OPTIMAL)
            .usage(vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT)
            .sharing_mode(vk::SharingMode::EXCLUSIVE)
            .queue_family_indices(&queuefamilies);
        let allocation_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::GpuOnly,
            ..Default::default()
        };
        let (depth_image, depth_image_allocation, depth_image_allocation_info) =
            allocator.create_image(&depth_image_info, &allocation_info)?;
        let subresource_range = vk::ImageSubresourceRange::builder()
            .aspect_mask(vk::ImageAspectFlags::DEPTH)
            .base_mip_level(0)
            .level_count(1)
            .base_array_layer(0)
            .layer_count(1);
        let imageview_create_info = vk::ImageViewCreateInfo::builder()
            .image(depth_image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::D32_SFLOAT)
            .subresource_range(*subresource_range);
        let depth_imageview =
            unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;

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
            depth_image,
            depth_image_allocation,
            depth_image_allocation_info,
            depth_imageview,
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
            let iview = [*iv, self.depth_imageview];
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
    unsafe fn cleanup(&mut self, logical_device: &ash::Device, allocator: &vk_mem::Allocator) {
        logical_device.destroy_image_view(self.depth_imageview, None);
        allocator.destroy_image(self.depth_image, &self.depth_image_allocation);
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
    format: vk::Format,
) -> Result<vk::RenderPass, vk::Result> {
    let attachments = [
        vk::AttachmentDescription::builder()
            .format(format)
            .load_op(vk::AttachmentLoadOp::CLEAR)
            .store_op(vk::AttachmentStoreOp::STORE)
            .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
            .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
            .samples(vk::SampleCountFlags::TYPE_1)
            .build(),
        vk::AttachmentDescription::builder()
            .format(vk::Format::D32_SFLOAT)
            .load_op(vk::AttachmentLoadOp::CLEAR)
            .store_op(vk::AttachmentStoreOp::DONT_CARE)
            .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
            .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
            .samples(vk::SampleCountFlags::TYPE_1)
            .build(),
    ];
    let color_attachment_references = [vk::AttachmentReference {
        attachment: 0,
        layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
    }];
    let depth_attachment_reference = vk::AttachmentReference {
        attachment: 1,
        layout: vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
    };
    let subpasses = [vk::SubpassDescription::builder()
        .color_attachments(&color_attachment_references)
        .depth_stencil_attachment(&depth_attachment_reference)
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
    descriptor_set_layouts: Vec<vk::DescriptorSetLayout>,
}

impl Pipeline {
    fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            for dsl in &self.descriptor_set_layouts {
                logical_device.destroy_descriptor_set_layout(*dsl, None);
            }
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
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 1,
                offset: 0,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 2,
                offset: 16,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 3,
                offset: 32,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 4,
                offset: 48,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 5,
                offset: 64,
                format: vk::Format::R32G32B32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 12,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 76,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
            .vertex_attribute_descriptions(&vertex_attrib_descs)
            .vertex_binding_descriptions(&vertex_binding_descs);
        let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::TRIANGLE_LIST);
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
        let depth_stencil_info = vk::PipelineDepthStencilStateCreateInfo::builder()
            .depth_test_enable(true)
            .depth_write_enable(true)
            .depth_compare_op(vk::CompareOp::LESS_OR_EQUAL);
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
        let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
            .descriptor_count(1)
            .stage_flags(vk::ShaderStageFlags::VERTEX)
            .build()];
        let descriptorset_layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs);
        let descriptorsetlayout = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info, None)
        }?;
        let desclayouts = vec![descriptorsetlayout];
        let pipelinelayout_info = vk::PipelineLayoutCreateInfo::builder().set_layouts(&desclayouts);
        let pipelinelayout =
            unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
        let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&shader_stages)
            .vertex_input_state(&vertex_input_info)
            .input_assembly_state(&input_assembly_info)
            .viewport_state(&viewport_info)
            .rasterization_state(&rasterizer_info)
            .multisample_state(&multisampler_info)
            .depth_stencil_state(&depth_stencil_info)
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
            descriptor_set_layouts: desclayouts,
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

struct Buffer {
    buffer: vk::Buffer,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
    size_in_bytes: u64,
    buffer_usage: vk::BufferUsageFlags,
    memory_usage: vk_mem::MemoryUsage,
}

impl Buffer {
    fn new(
        allocator: &vk_mem::Allocator,
        size_in_bytes: u64,
        buffer_usage: vk::BufferUsageFlags,
        memory_usage: vk_mem::MemoryUsage,
    ) -> Result<Buffer, vk_mem::error::Error> {
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: memory_usage,
            ..Default::default()
        };
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(size_in_bytes)
                .usage(buffer_usage)
                .build(),
            &allocation_create_info,
        )?;
        Ok(Buffer {
            buffer,
            allocation,
            allocation_info,
            size_in_bytes,
            buffer_usage,
            memory_usage,
        })
    }
    fn fill<T: Sized>(
        &mut self,
        allocator: &vk_mem::Allocator,
        data: &[T],
    ) -> Result<(), vk_mem::error::Error> {
        let bytes_to_write = (data.len() * std::mem::size_of::<T>()) as u64;
        if bytes_to_write > self.size_in_bytes {
            allocator.destroy_buffer(self.buffer, &self.allocation);
            let newbuffer = Buffer::new(
                allocator,
                bytes_to_write,
                self.buffer_usage,
                self.memory_usage,
            )?;
            *self = newbuffer;
        }
        let data_ptr = allocator.map_memory(&self.allocation)? as *mut T;
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), data.len()) };
        allocator.unmap_memory(&self.allocation)?;
        Ok(())
    }
}
#[derive(Debug, Clone)]
struct InvalidHandle;
impl std::fmt::Display for InvalidHandle {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "invalid handle")
    }
}
impl std::error::Error for InvalidHandle {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        None
    }
}

struct Model<V, I> {
    vertexdata: Vec<V>,
    indexdata: Vec<u32>,
    handle_to_index: std::collections::HashMap<usize, usize>,
    handles: Vec<usize>,
    instances: Vec<I>,
    first_invisible: usize,
    next_handle: usize,
    vertexbuffer: Option<Buffer>,
    indexbuffer: Option<Buffer>,
    instancebuffer: Option<Buffer>,
}
impl<V, I> Model<V, I> {
    fn get(&self, handle: usize) -> Option<&I> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            self.instances.get(index)
        } else {
            None
        }
    }
    fn get_mut(&mut self, handle: usize) -> Option<&mut I> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            self.instances.get_mut(index)
        } else {
            None
        }
    }
    fn is_visible(&self, handle: usize) -> Result<bool, InvalidHandle> {
        if let Some(index) = self.handle_to_index.get(&handle) {
            Ok(index < &self.first_invisible)
        } else {
            Err(InvalidHandle)
        }
    }
    fn make_visible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
        //if already visible: do nothing
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index < self.first_invisible {
                return Ok(());
            }
            //else: move to position first_invisible and increase value of first_invisible
            self.swap_by_index(index, self.first_invisible);
            self.first_invisible += 1;
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    fn make_invisible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
        //if already invisible: do nothing
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index >= self.first_invisible {
                return Ok(());
            }
            //else: move to position before first_invisible and decrease value of first_invisible
            self.swap_by_index(index, self.first_invisible - 1);
            self.first_invisible -= 1;
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    fn insert(&mut self, element: I) -> usize {
        let handle = self.next_handle;
        self.next_handle += 1;
        let index = self.instances.len();
        self.instances.push(element);
        self.handles.push(handle);
        self.handle_to_index.insert(handle, index);
        handle
    }
    fn insert_visibly(&mut self, element: I) -> usize {
        let new_handle = self.insert(element);
        self.make_visible(new_handle).ok();
        new_handle
    }
    fn remove(&mut self, handle: usize) -> Result<I, InvalidHandle> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index < self.first_invisible {
                self.swap_by_index(index, self.first_invisible - 1);
                self.first_invisible -= 1;
            }
            self.swap_by_index(self.first_invisible, self.instances.len() - 1);
            self.handles.pop();
            self.handle_to_index.remove(&handle);
            //must be Some(), otherwise we couldn't have found an index
            Ok(self.instances.pop().unwrap())
        } else {
            Err(InvalidHandle)
        }
    }
    fn swap_by_handle(&mut self, handle1: usize, handle2: usize) -> Result<(), InvalidHandle> {
        if handle1 == handle2 {
            return Ok(());
        }
        if let (Some(&index1), Some(&index2)) = (
            self.handle_to_index.get(&handle1),
            self.handle_to_index.get(&handle2),
        ) {
            self.handles.swap(index1, index2);
            self.instances.swap(index1, index2);
            self.handle_to_index.insert(index1, handle2);
            self.handle_to_index.insert(index2, handle1);
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    fn swap_by_index(&mut self, index1: usize, index2: usize) {
        if index1 == index2 {
            return;
        }
        let handle1 = self.handles[index1];
        let handle2 = self.handles[index2];
        self.handles.swap(index1, index2);
        self.instances.swap(index1, index2);
        self.handle_to_index.insert(index1, handle2);
        self.handle_to_index.insert(index2, handle1);
    }
    fn update_vertexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.vertexbuffer {
            buffer.fill(allocator, &self.vertexdata)?;
            Ok(())
        } else {
            let bytes = (self.vertexdata.len() * std::mem::size_of::<V>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::VERTEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.vertexdata)?;
            self.vertexbuffer = Some(buffer);
            Ok(())
        }
    }
    fn update_indexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.indexbuffer {
            buffer.fill(allocator, &self.indexdata)?;
            Ok(())
        } else {
            let bytes = (self.indexdata.len() * std::mem::size_of::<u32>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::INDEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.indexdata)?;
            self.indexbuffer = Some(buffer);
            Ok(())
        }
    }
    fn update_instancebuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.instancebuffer {
            buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
            Ok(())
        } else {
            let bytes = (self.first_invisible * std::mem::size_of::<I>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::VERTEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
            self.instancebuffer = Some(buffer);
            Ok(())
        }
    }
    fn draw(&self, logical_device: &ash::Device, commandbuffer: vk::CommandBuffer) {
        if let Some(vertexbuffer) = &self.vertexbuffer {
            if let Some(indexbuffer) = &self.indexbuffer {
                if let Some(instancebuffer) = &self.instancebuffer {
                    if self.first_invisible > 0 {
                        unsafe {
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                0,
                                &[vertexbuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                1,
                                &[instancebuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_index_buffer(
                                commandbuffer,
                                indexbuffer.buffer,
                                0,
                                vk::IndexType::UINT32,
                            );
                            logical_device.cmd_draw_indexed(
                                commandbuffer,
                                self.indexdata.len() as u32,
                                self.first_invisible as u32,
                                0,
                                0,
                                0,
                            );
                        }
                    }
                }
            }
        }
    }
}
impl Model<[f32; 3], InstanceData> {
    fn cube() -> Model<[f32; 3], InstanceData> {
        let lbf = [-1.0, 1.0, -1.0]; //lbf: left-bottom-front
        let lbb = [-1.0, 1.0, 1.0];
        let ltf = [-1.0, -1.0, -1.0];
        let ltb = [-1.0, -1.0, 1.0];
        let rbf = [1.0, 1.0, -1.0];
        let rbb = [1.0, 1.0, 1.0];
        let rtf = [1.0, -1.0, -1.0];
        let rtb = [1.0, -1.0, 1.0];
        Model {
            vertexdata: vec![lbf, lbb, ltf, ltb, rbf, rbb, rtf, rtb],
            indexdata: vec![
                0, 1, 5, 0, 5, 4, //bottom
                2, 7, 3, 2, 6, 7, //top
                0, 6, 2, 0, 4, 6, //front
                1, 3, 7, 1, 7, 5, //back
                0, 2, 1, 1, 2, 3, //left
                4, 5, 6, 5, 7, 6, //right
            ],
            handle_to_index: std::collections::HashMap::new(),
            handles: Vec::new(),
            instances: Vec::new(),
            first_invisible: 0,
            next_handle: 0,
            vertexbuffer: None,
            indexbuffer: None,
            instancebuffer: None,
        }
    }
}

#[repr(C)]
struct InstanceData {
    modelmatrix: [[f32; 4]; 4],
    colour: [f32; 3],
}

struct Camera {
    viewmatrix: na::Matrix4<f32>,
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
    projectionmatrix: na::Matrix4<f32>,
}
struct CameraBuilder {
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
}
impl CameraBuilder {
    fn build(self) -> Camera {
        if self.far < self.near {
            println!(
                "far plane (at {}) closer than near plane (at {}) — is that right?",
                self.far, self.near
            );
        }
        let mut cam = Camera {
            position: self.position,
            view_direction: self.view_direction,
            down_direction: na::Unit::new_normalize(
                self.down_direction.as_ref()
                    - self
                        .down_direction
                        .as_ref()
                        .dot(self.view_direction.as_ref())
                        * self.view_direction.as_ref(),
            ),
            fovy: self.fovy,
            aspect: self.aspect,
            near: self.near,
            far: self.far,
            viewmatrix: na::Matrix4::identity(),
            projectionmatrix: na::Matrix4::identity(),
        };
        cam.update_projectionmatrix();
        cam.update_viewmatrix();
        cam
    }
    fn position(mut self, pos: na::Vector3<f32>) -> CameraBuilder {
        self.position = pos;
        self
    }
    fn fovy(mut self, fovy: f32) -> CameraBuilder {
        self.fovy = fovy.max(0.01).min(std::f32::consts::PI - 0.01);
        self
    }
    fn aspect(mut self, aspect: f32) -> CameraBuilder {
        self.aspect = aspect;
        self
    }
    fn near(mut self, near: f32) -> CameraBuilder {
        if near <= 0.0 {
            println!("setting near plane to negative value: {} — you sure?", near);
        }
        self.near = near;
        self
    }
    fn far(mut self, far: f32) -> CameraBuilder {
        if far <= 0.0 {
            println!("setting far plane to negative value: {} — you sure?", far);
        }
        self.far = far;
        self
    }
    fn view_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.view_direction = na::Unit::new_normalize(direction);
        self
    }
    fn down_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.down_direction = na::Unit::new_normalize(direction);
        self
    }
}
impl Camera {
    fn builder() -> CameraBuilder {
        CameraBuilder {
            position: na::Vector3::new(0.0, -3.0, -3.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, -1.0)),
            fovy: std::f32::consts::FRAC_PI_3,
            aspect: 800.0 / 600.0,
            near: 0.1,
            far: 100.0,
        }
    }
    fn update_projectionmatrix(&mut self) {
        let d = 1.0 / (0.5 * self.fovy).tan();
        self.projectionmatrix = na::Matrix4::new(
            d / self.aspect,
            0.0,
            0.0,
            0.0,
            0.0,
            d,
            0.0,
            0.0,
            0.0,
            0.0,
            self.far / (self.far - self.near),
            -self.near * self.far / (self.far - self.near),
            0.0,
            0.0,
            1.0,
            0.0,
        );
    }
    fn update_buffer(&self, allocator: &vk_mem::Allocator, buffer: &mut Buffer) {
        let data: [[[f32; 4]; 4]; 2] = [self.viewmatrix.into(), self.projectionmatrix.into()];
        buffer.fill(allocator, &data);
    }
    fn update_viewmatrix(&mut self) {
        let right = na::Unit::new_normalize(self.down_direction.cross(&self.view_direction));
        let m = na::Matrix4::new(
            right.x,
            right.y,
            right.z,
            -right.dot(&self.position), //
            self.down_direction.x,
            self.down_direction.y,
            self.down_direction.z,
            -self.down_direction.dot(&self.position), //
            self.view_direction.x,
            self.view_direction.y,
            self.view_direction.z,
            -self.view_direction.dot(&self.position), //
            0.0,
            0.0,
            0.0,
            1.0,
        );
        self.viewmatrix = self.projectionmatrix * m;
    }
    fn move_forward(&mut self, distance: f32) {
        self.position += distance * self.view_direction.as_ref();
        self.update_viewmatrix();
    }
    fn move_backward(&mut self, distance: f32) {
        self.move_forward(-distance);
    }
    fn turn_right(&mut self, angle: f32) {
        let rotation = na::Rotation3::from_axis_angle(&self.down_direction, angle);
        self.view_direction = rotation * self.view_direction;
        self.update_viewmatrix();
    }
    fn turn_left(&mut self, angle: f32) {
        self.turn_right(-angle);
    }
    fn turn_up(&mut self, angle: f32) {
        let right = na::Unit::new_normalize(self.down_direction.cross(&self.view_direction));
        let rotation = na::Rotation3::from_axis_angle(&right, angle);
        self.view_direction = rotation * self.view_direction;
        self.down_direction = rotation * self.down_direction;
        self.update_viewmatrix();
    }
    fn turn_down(&mut self, angle: f32) {
        self.turn_up(-angle);
    }
}

struct Aetna {
    window: winit::window::Window,
    entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    physical_device: vk::PhysicalDevice,
    physical_device_properties: vk::PhysicalDeviceProperties,
    physical_device_features: vk::PhysicalDeviceFeatures,
    queue_families: QueueFamilies,
    queues: Queues,
    device: ash::Device,
    swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    commandbuffers: Vec<vk::CommandBuffer>,
    allocator: vk_mem::Allocator,
    models: Vec<Model<[f32; 3], InstanceData>>,
    uniformbuffer: Buffer,
    descriptor_pool: vk::DescriptorPool,
    descriptor_sets: Vec<vk::DescriptorSet>,
}

impl Aetna {
    fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties, physical_device_features) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;

        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let allocator = vk_mem::Allocator::new(&allocator_create_info)?;

        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &allocator,
        )?;
        let renderpass = init_renderpass(&logical_device, swapchain.surface_format.format)?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;

        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;

        let mut uniformbuffer = Buffer::new(
            &allocator,
            64,
            vk::BufferUsageFlags::UNIFORM_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        let cameratransforms: [[[f32; 4]; 4]; 2] = [
            na::Matrix4::identity().into(),
            na::Matrix4::identity().into(),
        ];
        uniformbuffer.fill(&allocator, &cameratransforms)?;
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::UNIFORM_BUFFER,
            descriptor_count: swapchain.amount_of_images,
        }];
        let descriptor_pool_info = vk::DescriptorPoolCreateInfo::builder()
            .max_sets(swapchain.amount_of_images)
            .pool_sizes(&pool_sizes);
        let descriptor_pool =
            unsafe { logical_device.create_descriptor_pool(&descriptor_pool_info, None) }?;

        let desc_layouts =
            vec![pipeline.descriptor_set_layouts[0]; swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts);
        let descriptor_sets =
            unsafe { logical_device.allocate_descriptor_sets(&descriptor_set_allocate_info) }?;

        for (i, descset) in descriptor_sets.iter().enumerate() {
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: uniformbuffer.buffer,
                offset: 0,
                range: 128,
            }];
            let desc_sets_write = [vk::WriteDescriptorSet::builder()
                .dst_set(*descset)
                .dst_binding(0)
                .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
                .buffer_info(&buffer_infos)
                .build()];
            unsafe { logical_device.update_descriptor_sets(&desc_sets_write, &[]) };
        }

        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            physical_device_features,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            models: vec![],
            uniformbuffer,
            descriptor_pool,
            descriptor_sets,
        })
    }
    fn update_commandbuffer(&mut self, index: usize) -> Result<(), vk::Result> {
        let commandbuffer = self.commandbuffers[index];
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            self.device
                .begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [
            vk::ClearValue {
                color: vk::ClearColorValue {
                    float32: [0.0, 0.0, 0.08, 1.0],
                },
            },
            vk::ClearValue {
                depth_stencil: vk::ClearDepthStencilValue {
                    depth: 1.0,
                    stencil: 0,
                },
            },
        ];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(self.renderpass)
            .framebuffer(self.swapchain.framebuffers[index])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: self.swapchain.extent,
            })
            .clear_values(&clearvalues);
        unsafe {
            self.device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            self.device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.pipeline,
            );
            self.device.cmd_bind_descriptor_sets(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.layout,
                0,
                &[self.descriptor_sets[index]],
                &[],
            );
            for m in &self.models {
                m.draw(&self.device, commandbuffer);
            }
            self.device.cmd_end_render_pass(commandbuffer);
            self.device.end_command_buffer(commandbuffer)?;
        }
        Ok(())
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.device
                .destroy_descriptor_pool(self.descriptor_pool, None);
            self.allocator
                .destroy_buffer(self.uniformbuffer.buffer, &self.uniformbuffer.allocation);
            for m in &self.models {
                if let Some(vb) = &m.vertexbuffer {
                    self.allocator
                        .destroy_buffer(vb.buffer, &vb.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.instancebuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.indexbuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
            }
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device, &self.allocator);
            self.allocator.destroy();
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
```
Let's see what we can separate. Are there parts of the code that are somehow a logical unit of their own? 

How about the `Camera`? Not quite the `Camera` struct alone, but maybe `Camera` and `CameraBuilder` together. They certainly belong together, and I
expect that the question "should this function be in the Camera module?" is a rather clear-cut case (for every function we'll come up with). 

As to the actual refactoring into separate files, I'm again and again pleasantly surprised how pleasant the process is in Rust. 

First step: We begin by writing 
```rust
mod camera {
```
in front of 
```rust
    struct Camera {
```
and close those opening braces after 
```rust
        fn turn_down(&mut self, angle: f32) {
            self.turn_up(-angle);
        }
    }
```
that is after `struct Camera`, `struct CameraBuilder` and their respective `impl`s. 

Then we ask the compiler what's wrong, and it comes back with 
```
error: aborting due to 35 previous errors
```
— fortunately, with some more details. The most common one I see here is 
```
error[E0433]: failed to resolve: use of undeclared type or module `na`
```
because the `use nalgebra as na;` does not extend into this new module. Well, let's copy that line into it, directly after `mod camera {`. 
```
error: aborting due to 2 previous errors
```
Wow, that's progress. One of those is again an unknown type in the new camera module: `Buffer` could be `crate::Buffer` or `ash::vk::Buffer`. It is
the former, and we add `use crate::Buffer`.

Now, only `Camera` is unknown where we construct the camera. Well, let's add `use camera::Camera;` (this time to the top of the file, not in the new
module).

Progress! A new error message: 
```
5    | use camera::Camera;
     |             ^^^^^^ this struct is private
```
Okay, next sub-step is inserting `pub` where needed. `pub struct Camera` sounds suitable. (No reason to keep it hidden inside its module, we want to
work with it.) 

Also: `pub fn builder`, in response to the next error, and `pub fn build`. And while we're at it: We certainly want all of the other methods of
`CameraBuilder` public, too. (That's what they were made for — and relying on the compiler would not show them, because some (all?) of them are unused at the
moment.) 

The compiler also cries about the `turn_xxx` and `move_xxx` functions and `update_buffer`. We make them `pub`. 

Progress again: new errors. 
```
error: type `camera::CameraBuilder` is private
```
and 
```
error[E0446]: private type `camera::CameraBuilder` in public interface
    --> src/main.rs:1370:9
     |
1295 |       struct CameraBuilder {
     |       - `camera::CameraBuilder` declared as private
...
1370 | /         pub fn builder() -> CameraBuilder {
1371 | |             CameraBuilder {
1372 | |                 position: na::Vector3::new(0.0, -3.0, -3.0),
1373 | |                 view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 1.0)),
...    |
1379 | |             }
1380 | |         }
     | |_________^ can't leak private type
```
Right. I've made only the functions, not the struct public. Well, easy enough to change: `pub struct CameraBuilder`. There, done. Happy now, compiler?
Compiler? 
```
    Finished dev [unoptimized + debuginfo] target(s) in 4.89s
```
Okay. Good.

Next step: Take the whole `mod camera { /*...*/}` and cut and paste it into a new file. Remove the first line `mod camera {` and the last line `}`
from this file, save it as `camera.rs`, insert `mod camera;` in `main.rs`. Done.

What next? Maybe the debug stuff: 
```rust
mod debug {
    use ash::vk;
    pub struct DebugDongXi {
        loader: ash::extensions::ext::DebugUtils,
        messenger: vk::DebugUtilsMessengerEXT,
    }
    impl DebugDongXi {
        pub fn init(
            entry: &ash::Entry,
            instance: &ash::Instance,
        ) -> Result<DebugDongXi, vk::Result> {
            let debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
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

    pub unsafe extern "system" fn vulkan_debug_utils_callback(
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
}
```
`main.rs` needs two quick adjustments: `use debug::DebugDongXi;` and `.pfn_user_callback(Some(crate::debug::vulkan_debug_utils_callback));` in `init_instance`. 

We then create some `debug.rs` with this content (except first and last line) and with an additional `mod debug;` in main, we're ready to continue. 

```rust
mod surface {
    use ash::vk;
    pub struct SurfaceDongXi {
        xlib_surface_loader: ash::extensions::khr::XlibSurface,
        pub surface: vk::SurfaceKHR,
        surface_loader: ash::extensions::khr::Surface,
    }

    impl SurfaceDongXi {
        pub fn init(
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
            let surface =
                unsafe { xlib_surface_loader.create_xlib_surface(&x11_create_info, None) }?;
            let surface_loader = ash::extensions::khr::Surface::new(entry, instance);
            Ok(SurfaceDongXi {
                xlib_surface_loader,
                surface,
                surface_loader,
            })
        }
        pub fn get_capabilities(
            &self,
            physical_device: vk::PhysicalDevice,
        ) -> Result<vk::SurfaceCapabilitiesKHR, vk::Result> {
            unsafe {
                self.surface_loader
                    .get_physical_device_surface_capabilities(physical_device, self.surface)
            }
        }
        pub fn get_present_modes(
            &self,
            physical_device: vk::PhysicalDevice,
        ) -> Result<Vec<vk::PresentModeKHR>, vk::Result> {
            unsafe {
                self.surface_loader
                    .get_physical_device_surface_present_modes(physical_device, self.surface)
            }
        }
        pub fn get_formats(
            &self,
            physical_device: vk::PhysicalDevice,
        ) -> Result<Vec<vk::SurfaceFormatKHR>, vk::Result> {
            unsafe {
                self.surface_loader
                    .get_physical_device_surface_formats(physical_device, self.surface)
            }
        }
        pub fn get_physical_device_surface_support(
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
}
```

```rust
mod swapchain {
    use crate::surface::SurfaceDongXi;
    use crate::QueueFamilies;
    use ash::version::DeviceV1_0;
    use ash::vk;
    pub struct SwapchainDongXi {
        pub swapchain_loader: ash::extensions::khr::Swapchain,
        pub swapchain: vk::SwapchainKHR,
        pub images: Vec<vk::Image>,
        pub imageviews: Vec<vk::ImageView>,
        pub depth_image: vk::Image,
        depth_image_allocation: vk_mem::Allocation,
        depth_image_allocation_info: vk_mem::AllocationInfo,
        pub depth_imageview: vk::ImageView,
        pub framebuffers: Vec<vk::Framebuffer>,
        pub surface_format: vk::SurfaceFormatKHR,
        pub extent: vk::Extent2D,
        pub image_available: Vec<vk::Semaphore>,
        pub rendering_finished: Vec<vk::Semaphore>,
        pub may_begin_drawing: Vec<vk::Fence>,
        pub amount_of_images: u32,
        pub current_image: usize,
    }

    impl SwapchainDongXi {
        pub fn init(
            instance: &ash::Instance,
            physical_device: vk::PhysicalDevice,
            logical_device: &ash::Device,
            surfaces: &SurfaceDongXi,
            queue_families: &QueueFamilies,
            allocator: &vk_mem::Allocator,
        ) -> Result<SwapchainDongXi, Box<dyn std::error::Error>> {
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
            let swapchain =
                unsafe { swapchain_loader.create_swapchain(&swapchain_create_info, None)? };
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
            let extent3d = vk::Extent3D {
                width: extent.width,
                height: extent.height,
                depth: 1,
            };
            let depth_image_info = vk::ImageCreateInfo::builder()
                .image_type(vk::ImageType::TYPE_2D)
                .format(vk::Format::D32_SFLOAT)
                .extent(extent3d)
                .mip_levels(1)
                .array_layers(1)
                .samples(vk::SampleCountFlags::TYPE_1)
                .tiling(vk::ImageTiling::OPTIMAL)
                .usage(vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT)
                .sharing_mode(vk::SharingMode::EXCLUSIVE)
                .queue_family_indices(&queuefamilies);
            let allocation_info = vk_mem::AllocationCreateInfo {
                usage: vk_mem::MemoryUsage::GpuOnly,
                ..Default::default()
            };
            let (depth_image, depth_image_allocation, depth_image_allocation_info) =
                allocator.create_image(&depth_image_info, &allocation_info)?;
            let subresource_range = vk::ImageSubresourceRange::builder()
                .aspect_mask(vk::ImageAspectFlags::DEPTH)
                .base_mip_level(0)
                .level_count(1)
                .base_array_layer(0)
                .layer_count(1);
            let imageview_create_info = vk::ImageViewCreateInfo::builder()
                .image(depth_image)
                .view_type(vk::ImageViewType::TYPE_2D)
                .format(vk::Format::D32_SFLOAT)
                .subresource_range(*subresource_range);
            let depth_imageview =
                unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;

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
                depth_image,
                depth_image_allocation,
                depth_image_allocation_info,
                depth_imageview,
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
        pub fn create_framebuffers(
            &mut self,
            logical_device: &ash::Device,
            renderpass: vk::RenderPass,
        ) -> Result<(), vk::Result> {
            for iv in &self.imageviews {
                let iview = [*iv, self.depth_imageview];
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
        pub unsafe fn cleanup(
            &mut self,
            logical_device: &ash::Device,
            allocator: &vk_mem::Allocator,
        ) {
            logical_device.destroy_image_view(self.depth_imageview, None);
            allocator.destroy_image(self.depth_image, &self.depth_image_allocation);
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
}
```
And, of course, that becomes `swapchain.rs`.

Another one, heavily guided by the compiler's hints. Queues go with their families and hand in hand with the device; I don't want `init_instance`
completely on its own, it seems to fit here. (Not completely sure, might change that later.)

```rust
use instance_device_queues::{
    init_device_and_queues, init_instance, init_physical_device_and_properties, QueueFamilies,
    Queues,
};

mod instance_device_queues {
    use crate::surface::SurfaceDongXi;
    use ash::version::DeviceV1_0;
    use ash::version::EntryV1_0;
    use ash::version::InstanceV1_0;
    use ash::vk;
    pub fn init_instance(
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
            .pfn_user_callback(Some(crate::debug::vulkan_debug_utils_callback));

        let instance_create_info = vk::InstanceCreateInfo::builder()
            .push_next(&mut debugcreateinfo)
            .application_info(&app_info)
            .enabled_layer_names(&layer_name_pointers)
            .enabled_extension_names(&extension_name_pointers);
        unsafe { entry.create_instance(&instance_create_info, None) }
    }

    pub fn init_physical_device_and_properties(
        instance: &ash::Instance,
    ) -> Result<
        (
            vk::PhysicalDevice,
            vk::PhysicalDeviceProperties,
            vk::PhysicalDeviceFeatures,
        ),
        vk::Result,
    > {
        let phys_devs = unsafe { instance.enumerate_physical_devices()? };
        let mut chosen = None;
        for p in phys_devs {
            let properties = unsafe { instance.get_physical_device_properties(p) };
            let features = unsafe { instance.get_physical_device_features(p) };
            if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
                chosen = Some((p, properties, features));
            }
        }
        Ok(chosen.unwrap())
    }

    pub struct QueueFamilies {
        pub graphics_q_index: Option<u32>,
        pub transfer_q_index: Option<u32>,
    }
    impl QueueFamilies {
        pub fn init(
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

    pub struct Queues {
        pub graphics_queue: vk::Queue,
        pub transfer_queue: vk::Queue,
    }

    pub fn init_device_and_queues(
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
        let features = vk::PhysicalDeviceFeatures::builder().fill_mode_non_solid(true);
        let device_create_info = vk::DeviceCreateInfo::builder()
            .queue_create_infos(&queue_infos)
            .enabled_extension_names(&device_extension_name_pointers)
            .enabled_layer_names(&layer_name_pointers)
            .enabled_features(&features);
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
}
```
and another one (renderpass and pipeline seem to fit together rather well, conceptually): 
```rust
mod renderpass_and_pipeline {
    use crate::swapchain::SwapchainDongXi;
    use ash::version::DeviceV1_0;
    use ash::vk;
    pub fn init_renderpass(
        logical_device: &ash::Device,
        format: vk::Format,
    ) -> Result<vk::RenderPass, vk::Result> {
        let attachments = [
            vk::AttachmentDescription::builder()
                .format(format)
                .load_op(vk::AttachmentLoadOp::CLEAR)
                .store_op(vk::AttachmentStoreOp::STORE)
                .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
                .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
                .initial_layout(vk::ImageLayout::UNDEFINED)
                .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
                .samples(vk::SampleCountFlags::TYPE_1)
                .build(),
            vk::AttachmentDescription::builder()
                .format(vk::Format::D32_SFLOAT)
                .load_op(vk::AttachmentLoadOp::CLEAR)
                .store_op(vk::AttachmentStoreOp::DONT_CARE)
                .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
                .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
                .initial_layout(vk::ImageLayout::UNDEFINED)
                .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
                .samples(vk::SampleCountFlags::TYPE_1)
                .build(),
        ];
        let color_attachment_references = [vk::AttachmentReference {
            attachment: 0,
            layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
        }];
        let depth_attachment_reference = vk::AttachmentReference {
            attachment: 1,
            layout: vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
        };
        let subpasses = [vk::SubpassDescription::builder()
            .color_attachments(&color_attachment_references)
            .depth_stencil_attachment(&depth_attachment_reference)
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

    pub struct Pipeline {
        pub pipeline: vk::Pipeline,
        pub layout: vk::PipelineLayout,
        pub descriptor_set_layouts: Vec<vk::DescriptorSetLayout>,
    }

    impl Pipeline {
        pub fn cleanup(&self, logical_device: &ash::Device) {
            unsafe {
                for dsl in &self.descriptor_set_layouts {
                    logical_device.destroy_descriptor_set_layout(*dsl, None);
                }
                logical_device.destroy_pipeline(self.pipeline, None);
                logical_device.destroy_pipeline_layout(self.layout, None);
            }
        }

        pub fn init(
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
            let vertex_attrib_descs = [
                vk::VertexInputAttributeDescription {
                    binding: 0,
                    location: 0,
                    offset: 0,
                    format: vk::Format::R32G32B32_SFLOAT,
                },
                vk::VertexInputAttributeDescription {
                    binding: 1,
                    location: 1,
                    offset: 0,
                    format: vk::Format::R32G32B32A32_SFLOAT,
                },
                vk::VertexInputAttributeDescription {
                    binding: 1,
                    location: 2,
                    offset: 16,
                    format: vk::Format::R32G32B32A32_SFLOAT,
                },
                vk::VertexInputAttributeDescription {
                    binding: 1,
                    location: 3,
                    offset: 32,
                    format: vk::Format::R32G32B32A32_SFLOAT,
                },
                vk::VertexInputAttributeDescription {
                    binding: 1,
                    location: 4,
                    offset: 48,
                    format: vk::Format::R32G32B32A32_SFLOAT,
                },
                vk::VertexInputAttributeDescription {
                    binding: 1,
                    location: 5,
                    offset: 64,
                    format: vk::Format::R32G32B32_SFLOAT,
                },
            ];
            let vertex_binding_descs = [
                vk::VertexInputBindingDescription {
                    binding: 0,
                    stride: 12,
                    input_rate: vk::VertexInputRate::VERTEX,
                },
                vk::VertexInputBindingDescription {
                    binding: 1,
                    stride: 76,
                    input_rate: vk::VertexInputRate::INSTANCE,
                },
            ];
            let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
                .vertex_attribute_descriptions(&vertex_attrib_descs)
                .vertex_binding_descriptions(&vertex_binding_descs);
            let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
                .topology(vk::PrimitiveTopology::TRIANGLE_LIST);
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
            let depth_stencil_info = vk::PipelineDepthStencilStateCreateInfo::builder()
                .depth_test_enable(true)
                .depth_write_enable(true)
                .depth_compare_op(vk::CompareOp::LESS_OR_EQUAL);
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
            let colourblend_info = vk::PipelineColorBlendStateCreateInfo::builder()
                .attachments(&colourblend_attachments);
            let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
                .binding(0)
                .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
                .descriptor_count(1)
                .stage_flags(vk::ShaderStageFlags::VERTEX)
                .build()];
            let descriptorset_layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
                .bindings(&descriptorset_layout_binding_descs);
            let descriptorsetlayout = unsafe {
                logical_device.create_descriptor_set_layout(&descriptorset_layout_info, None)
            }?;
            let desclayouts = vec![descriptorsetlayout];
            let pipelinelayout_info =
                vk::PipelineLayoutCreateInfo::builder().set_layouts(&desclayouts);
            let pipelinelayout =
                unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
            let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
                .stages(&shader_stages)
                .vertex_input_state(&vertex_input_info)
                .input_assembly_state(&input_assembly_info)
                .viewport_state(&viewport_info)
                .rasterization_state(&rasterizer_info)
                .multisample_state(&multisampler_info)
                .depth_stencil_state(&depth_stencil_info)
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
                descriptor_set_layouts: desclayouts,
            })
        }
    }
}
```
yet another one (I don't like it much, but I'm not seeing much better places):
```rust
mod pools_and_commandbuffers {
    use crate::instance_device_queues::QueueFamilies;
    use ash::version::DeviceV1_0;
    use ash::vk;
    pub struct Pools {
        commandpool_graphics: vk::CommandPool,
        commandpool_transfer: vk::CommandPool,
    }

    impl Pools {
        pub fn init(
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
        pub fn cleanup(&self, logical_device: &ash::Device) {
            unsafe {
                logical_device.destroy_command_pool(self.commandpool_graphics, None);
                logical_device.destroy_command_pool(self.commandpool_transfer, None);
            }
        }
    }

    pub fn create_commandbuffers(
        logical_device: &ash::Device,
        pools: &Pools,
        amount: u32,
    ) -> Result<Vec<vk::CommandBuffer>, vk::Result> {
        let commandbuf_allocate_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(pools.commandpool_graphics)
            .command_buffer_count(amount);
        unsafe { logical_device.allocate_command_buffers(&commandbuf_allocate_info) }
    }
}
```
The next one I'm more convinced of: 
```rust
mod buffer {
    use ash::vk;
    pub struct Buffer {
        pub buffer: vk::Buffer,
        pub allocation: vk_mem::Allocation,
        allocation_info: vk_mem::AllocationInfo,
        pub size_in_bytes: u64,
        buffer_usage: vk::BufferUsageFlags,
        memory_usage: vk_mem::MemoryUsage,
    }

    impl Buffer {
        pub fn new(
            allocator: &vk_mem::Allocator,
            size_in_bytes: u64,
            buffer_usage: vk::BufferUsageFlags,
            memory_usage: vk_mem::MemoryUsage,
        ) -> Result<Buffer, vk_mem::error::Error> {
            let allocation_create_info = vk_mem::AllocationCreateInfo {
                usage: memory_usage,
                ..Default::default()
            };
            let (buffer, allocation, allocation_info) = allocator.create_buffer(
                &ash::vk::BufferCreateInfo::builder()
                    .size(size_in_bytes)
                    .usage(buffer_usage)
                    .build(),
                &allocation_create_info,
            )?;
            Ok(Buffer {
                buffer,
                allocation,
                allocation_info,
                size_in_bytes,
                buffer_usage,
                memory_usage,
            })
        }
        pub fn fill<T: Sized>(
            &mut self,
            allocator: &vk_mem::Allocator,
            data: &[T],
        ) -> Result<(), vk_mem::error::Error> {
            let bytes_to_write = (data.len() * std::mem::size_of::<T>()) as u64;
            if bytes_to_write > self.size_in_bytes {
                allocator.destroy_buffer(self.buffer, &self.allocation);
                let newbuffer = Buffer::new(
                    allocator,
                    bytes_to_write,
                    self.buffer_usage,
                    self.memory_usage,
                )?;
                *self = newbuffer;
            }
            let data_ptr = allocator.map_memory(&self.allocation)? as *mut T;
            unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), data.len()) };
            allocator.unmap_memory(&self.allocation)?;
            Ok(())
        }
    }
}
```
And `model.rs`:
```rust
    use crate::buffer::Buffer;
    use ash::version::DeviceV1_0;
    use ash::vk;

    #[derive(Debug, Clone)]
    struct InvalidHandle;
    impl std::fmt::Display for InvalidHandle {
        fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
            write!(f, "invalid handle")
        }
    }
    impl std::error::Error for InvalidHandle {
        fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
            None
        }
    }

    pub struct Model<V, I> {
        vertexdata: Vec<V>,
        indexdata: Vec<u32>,
        handle_to_index: std::collections::HashMap<usize, usize>,
        handles: Vec<usize>,
        instances: Vec<I>,
        first_invisible: usize,
        next_handle: usize,
        pub vertexbuffer: Option<Buffer>,
        pub indexbuffer: Option<Buffer>,
        pub instancebuffer: Option<Buffer>,
    }
    impl<V, I> Model<V, I> {
        pub fn get(&self, handle: usize) -> Option<&I> {
            if let Some(&index) = self.handle_to_index.get(&handle) {
                self.instances.get(index)
            } else {
                None
            }
        }
        pub fn get_mut(&mut self, handle: usize) -> Option<&mut I> {
            if let Some(&index) = self.handle_to_index.get(&handle) {
                self.instances.get_mut(index)
            } else {
                None
            }
        }
        pub fn is_visible(&self, handle: usize) -> Result<bool, InvalidHandle> {
            if let Some(index) = self.handle_to_index.get(&handle) {
                Ok(index < &self.first_invisible)
            } else {
                Err(InvalidHandle)
            }
        }
        pub fn make_visible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
            //if already visible: do nothing
            if let Some(&index) = self.handle_to_index.get(&handle) {
                if index < self.first_invisible {
                    return Ok(());
                }
                //else: move to position first_invisible and increase value of first_invisible
                self.swap_by_index(index, self.first_invisible);
                self.first_invisible += 1;
                Ok(())
            } else {
                Err(InvalidHandle)
            }
        }
        pub fn make_invisible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
            //if already invisible: do nothing
            if let Some(&index) = self.handle_to_index.get(&handle) {
                if index >= self.first_invisible {
                    return Ok(());
                }
                //else: move to position before first_invisible and decrease value of first_invisible
                self.swap_by_index(index, self.first_invisible - 1);
                self.first_invisible -= 1;
                Ok(())
            } else {
                Err(InvalidHandle)
            }
        }
        pub fn insert(&mut self, element: I) -> usize {
            let handle = self.next_handle;
            self.next_handle += 1;
            let index = self.instances.len();
            self.instances.push(element);
            self.handles.push(handle);
            self.handle_to_index.insert(handle, index);
            handle
        }
        pub fn insert_visibly(&mut self, element: I) -> usize {
            let new_handle = self.insert(element);
            self.make_visible(new_handle).ok();
            new_handle
        }
        pub fn remove(&mut self, handle: usize) -> Result<I, InvalidHandle> {
            if let Some(&index) = self.handle_to_index.get(&handle) {
                if index < self.first_invisible {
                    self.swap_by_index(index, self.first_invisible - 1);
                    self.first_invisible -= 1;
                }
                self.swap_by_index(self.first_invisible, self.instances.len() - 1);
                self.handles.pop();
                self.handle_to_index.remove(&handle);
                //must be Some(), otherwise we couldn't have found an index
                Ok(self.instances.pop().unwrap())
            } else {
                Err(InvalidHandle)
            }
        }
        pub fn swap_by_handle(
            &mut self,
            handle1: usize,
            handle2: usize,
        ) -> Result<(), InvalidHandle> {
            if handle1 == handle2 {
                return Ok(());
            }
            if let (Some(&index1), Some(&index2)) = (
                self.handle_to_index.get(&handle1),
                self.handle_to_index.get(&handle2),
            ) {
                self.handles.swap(index1, index2);
                self.instances.swap(index1, index2);
                self.handle_to_index.insert(index1, handle2);
                self.handle_to_index.insert(index2, handle1);
                Ok(())
            } else {
                Err(InvalidHandle)
            }
        }
        fn swap_by_index(&mut self, index1: usize, index2: usize) {
            if index1 == index2 {
                return;
            }
            let handle1 = self.handles[index1];
            let handle2 = self.handles[index2];
            self.handles.swap(index1, index2);
            self.instances.swap(index1, index2);
            self.handle_to_index.insert(index1, handle2);
            self.handle_to_index.insert(index2, handle1);
        }
        pub fn update_vertexbuffer(
            &mut self,
            allocator: &vk_mem::Allocator,
        ) -> Result<(), vk_mem::error::Error> {
            if let Some(buffer) = &mut self.vertexbuffer {
                buffer.fill(allocator, &self.vertexdata)?;
                Ok(())
            } else {
                let bytes = (self.vertexdata.len() * std::mem::size_of::<V>()) as u64;
                let mut buffer = Buffer::new(
                    &allocator,
                    bytes,
                    vk::BufferUsageFlags::VERTEX_BUFFER,
                    vk_mem::MemoryUsage::CpuToGpu,
                )?;
                buffer.fill(allocator, &self.vertexdata)?;
                self.vertexbuffer = Some(buffer);
                Ok(())
            }
        }
        pub fn update_indexbuffer(
            &mut self,
            allocator: &vk_mem::Allocator,
        ) -> Result<(), vk_mem::error::Error> {
            if let Some(buffer) = &mut self.indexbuffer {
                buffer.fill(allocator, &self.indexdata)?;
                Ok(())
            } else {
                let bytes = (self.indexdata.len() * std::mem::size_of::<u32>()) as u64;
                let mut buffer = Buffer::new(
                    &allocator,
                    bytes,
                    vk::BufferUsageFlags::INDEX_BUFFER,
                    vk_mem::MemoryUsage::CpuToGpu,
                )?;
                buffer.fill(allocator, &self.indexdata)?;
                self.indexbuffer = Some(buffer);
                Ok(())
            }
        }
        pub fn update_instancebuffer(
            &mut self,
            allocator: &vk_mem::Allocator,
        ) -> Result<(), vk_mem::error::Error> {
            if let Some(buffer) = &mut self.instancebuffer {
                buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
                Ok(())
            } else {
                let bytes = (self.first_invisible * std::mem::size_of::<I>()) as u64;
                let mut buffer = Buffer::new(
                    &allocator,
                    bytes,
                    vk::BufferUsageFlags::VERTEX_BUFFER,
                    vk_mem::MemoryUsage::CpuToGpu,
                )?;
                buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
                self.instancebuffer = Some(buffer);
                Ok(())
            }
        }
        pub fn draw(&self, logical_device: &ash::Device, commandbuffer: vk::CommandBuffer) {
            if let Some(vertexbuffer) = &self.vertexbuffer {
                if let Some(indexbuffer) = &self.indexbuffer {
                    if let Some(instancebuffer) = &self.instancebuffer {
                        if self.first_invisible > 0 {
                            unsafe {
                                logical_device.cmd_bind_vertex_buffers(
                                    commandbuffer,
                                    0,
                                    &[vertexbuffer.buffer],
                                    &[0],
                                );
                                logical_device.cmd_bind_vertex_buffers(
                                    commandbuffer,
                                    1,
                                    &[instancebuffer.buffer],
                                    &[0],
                                );
                                logical_device.cmd_bind_index_buffer(
                                    commandbuffer,
                                    indexbuffer.buffer,
                                    0,
                                    vk::IndexType::UINT32,
                                );
                                logical_device.cmd_draw_indexed(
                                    commandbuffer,
                                    self.indexdata.len() as u32,
                                    self.first_invisible as u32,
                                    0,
                                    0,
                                    0,
                                );
                            }
                        }
                    }
                }
            }
        }
    }
    impl Model<[f32; 3], InstanceData> {
        pub fn cube() -> Model<[f32; 3], InstanceData> {
            let lbf = [-1.0, 1.0, -1.0]; //lbf: left-bottom-front
            let lbb = [-1.0, 1.0, 1.0];
            let ltf = [-1.0, -1.0, -1.0];
            let ltb = [-1.0, -1.0, 1.0];
            let rbf = [1.0, 1.0, -1.0];
            let rbb = [1.0, 1.0, 1.0];
            let rtf = [1.0, -1.0, -1.0];
            let rtb = [1.0, -1.0, 1.0];
            Model {
                vertexdata: vec![lbf, lbb, ltf, ltb, rbf, rbb, rtf, rtb],
                indexdata: vec![
                    0, 1, 5, 0, 5, 4, //bottom
                    2, 7, 3, 2, 6, 7, //top
                    0, 6, 2, 0, 4, 6, //front
                    1, 3, 7, 1, 7, 5, //back
                    0, 2, 1, 1, 2, 3, //left
                    4, 5, 6, 5, 7, 6, //right
                ],
                handle_to_index: std::collections::HashMap::new(),
                handles: Vec::new(),
                instances: Vec::new(),
                first_invisible: 0,
                next_handle: 0,
                vertexbuffer: None,
                indexbuffer: None,
                instancebuffer: None,
            }
        }
    }

    #[repr(C)]
    pub struct InstanceData {
        pub modelmatrix: [[f32; 4]; 4],
        pub colour: [f32; 3],
    }
```
Now, not much is left 

```rust
mod aetna{
use crate::*;
use ash::version::DeviceV1_0;
use ash::version::InstanceV1_0;
use ash::vk;

pub struct Aetna {
    pub window: winit::window::Window,
    entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    physical_device: vk::PhysicalDevice,
    physical_device_properties: vk::PhysicalDeviceProperties,
    physical_device_features: vk::PhysicalDeviceFeatures,
    pub queue_families: QueueFamilies,
    pub queues: Queues,
    pub device: ash::Device,
    pub swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    pub commandbuffers: Vec<vk::CommandBuffer>,
    pub allocator: vk_mem::Allocator,
    pub models: Vec<Model<[f32; 3], InstanceData>>,
    pub uniformbuffer: Buffer,
    descriptor_pool: vk::DescriptorPool,
    descriptor_sets: Vec<vk::DescriptorSet>,
}

impl Aetna {
    pub fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties, physical_device_features) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;

        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let allocator = vk_mem::Allocator::new(&allocator_create_info)?;

        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &allocator,
        )?;
        let renderpass = init_renderpass(&logical_device, swapchain.surface_format.format)?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;

        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;

        let mut uniformbuffer = Buffer::new(
            &allocator,
            64,
            vk::BufferUsageFlags::UNIFORM_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        let cameratransforms: [[[f32; 4]; 4]; 2] = [
            na::Matrix4::identity().into(),
            na::Matrix4::identity().into(),
        ];
        uniformbuffer.fill(&allocator, &cameratransforms)?;
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::UNIFORM_BUFFER,
            descriptor_count: swapchain.amount_of_images,
        }];
        let descriptor_pool_info = vk::DescriptorPoolCreateInfo::builder()
            .max_sets(swapchain.amount_of_images)
            .pool_sizes(&pool_sizes);
        let descriptor_pool =
            unsafe { logical_device.create_descriptor_pool(&descriptor_pool_info, None) }?;

        let desc_layouts =
            vec![pipeline.descriptor_set_layouts[0]; swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts);
        let descriptor_sets =
            unsafe { logical_device.allocate_descriptor_sets(&descriptor_set_allocate_info) }?;

        for (i, descset) in descriptor_sets.iter().enumerate() {
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: uniformbuffer.buffer,
                offset: 0,
                range: 128,
            }];
            let desc_sets_write = [vk::WriteDescriptorSet::builder()
                .dst_set(*descset)
                .dst_binding(0)
                .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
                .buffer_info(&buffer_infos)
                .build()];
            unsafe { logical_device.update_descriptor_sets(&desc_sets_write, &[]) };
        }

        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            physical_device_features,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            models: vec![],
            uniformbuffer,
            descriptor_pool,
            descriptor_sets,
        })
    }
    pub fn update_commandbuffer(&mut self, index: usize) -> Result<(), vk::Result> {
        let commandbuffer = self.commandbuffers[index];
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            self.device
                .begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [
            vk::ClearValue {
                color: vk::ClearColorValue {
                    float32: [0.0, 0.0, 0.08, 1.0],
                },
            },
            vk::ClearValue {
                depth_stencil: vk::ClearDepthStencilValue {
                    depth: 1.0,
                    stencil: 0,
                },
            },
        ];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(self.renderpass)
            .framebuffer(self.swapchain.framebuffers[index])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: self.swapchain.extent,
            })
            .clear_values(&clearvalues);
        unsafe {
            self.device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            self.device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.pipeline,
            );
            self.device.cmd_bind_descriptor_sets(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.layout,
                0,
                &[self.descriptor_sets[index]],
                &[],
            );
            for m in &self.models {
                m.draw(&self.device, commandbuffer);
            }
            self.device.cmd_end_render_pass(commandbuffer);
            self.device.end_command_buffer(commandbuffer)?;
        }
        Ok(())
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.device
                .destroy_descriptor_pool(self.descriptor_pool, None);
            self.allocator
                .destroy_buffer(self.uniformbuffer.buffer, &self.uniformbuffer.allocation);
            for m in &self.models {
                if let Some(vb) = &m.vertexbuffer {
                    self.allocator
                        .destroy_buffer(vb.buffer, &vb.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.instancebuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.indexbuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
            }
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device, &self.allocator);
            self.allocator.destroy();
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
}
```
Worth noting is the first line: In case we don't want to import everything separately. I don't recommend using `use xy::*` for every crate (too easy
to get lost in "what did I mean, where did that struct come from?"), but for the own crate it seems good.

With that, only `main()` remains in `main.rs`. 

All of the changes were straightforward. The only thing to note about the compiler messages is that 
```
    = note: the following trait is implemented but not in scope; perhaps add a `use` for it:
            `use ash::device::DeviceV1_0;`
```
does not require the inclusion of `use ash::device::DeviceV1_0`, but rather of `use ash::version::DeviceV1_0` (as the non-private variant). 


Okay. On to the output. There are warnings for us.

Like 
```
warning: unused import: `ash::version::EntryV1_0`
```
That's an outcome of the (apparently unfinished) cleanup. But easy to solve: I'll just delete this line.

```
warning: unused variable: `i`
   --> src/aetna.rs:100:14
    |
100 |         for (i, descset) in descriptor_sets.iter().enumerate() {
    |              ^ help: consider prefixing with an underscore: `_i`
    |
    = note: `#[warn(unused_variables)]` on by default
```
Okay, if I'm not using `i` anyways, I can remove the `enumerate()` as well: `for descset in &descriptor_sets {`. 

```
warning: unused variable: `surface_present_modes`
  --> src/swapchain.rs:35:13
```
Okay, that line can become a comment. (I don't want to remove it, so that I don't completely forget about this function; but for the choice of present
mode FIFO I don't need it (FIFO is required to be supported).)

```
warning: private type `model::InvalidHandle` in public interface (error E0446)
```
for several lines in `model.rs`. Seems someone has forgotten about `pub` in `pub struct InvalidHandle;`. — Who was supposed to clean up?

Then we have some 
```
warning: field is never read: `entry`
```
for fields of `Aetna`. Some of them (like `entry`) absolutely have to be kept around for the whole duration of the program. I'll give their names an
initial underscore `_entry`, that will silence the warning.

```
warning: field is never read: `allocation_info`
 --> src/buffer.rs:5:5
```
Well, same treatment. (There must be some reason that `vk_mem` always returns this allocation info, better keep it, it might come up later.) 

```
warning: method is never used: `position`
```
... and the other methods of `CameraBuilder`. Well, that's okay. I don't want to get rid of them, maybe one day we'll use a less restricted variant of
the camera; I'm quite happy to have these. 
```rust
#[allow(dead_code)]
impl CameraBuilder {
```
Now also the compiler knows that, and won't tell us about these functions. 

Similar warnings concern methods of `Model`. Well, I know we aren't using all of them, but I have thought about those methods when inventing `Model`,
I want to keep them (and expect to use them much later). That's another `#[allow(dead_code)]`:
```rust
#[allow(dead_code)]
impl<V, I> Model<V, I> {
```

```
warning: field is never read: `xlib_surface_loader`
 --> src/surface.rs:3:5
```
Another underscore, I guess. 

```
warning: method is never used: `get_present_modes`
  --> src/surface.rs:38:5
```
Wait. Wasn't that a function we have just recently removed, because the result was not entirely necessary, but which I still wanted to keep around?
Then let's not make that line in `swapchain.rs` a comment, but rather prefix the variable with an underscore:
```rust
        let _surface_present_modes = surfaces.get_present_modes(physical_device)?;
```
Next error: 
```
warning: field is never read: `depth_image_allocation_info`
```
Okay, another underscore. I'm really wary of just deleting this allocation info.

The remaining warnings are 
```
warning: unused `std::result::Result` that must be used
```
Eight times. That is: eight missing question marks.

```
error[E0277]: the `?` operator can only be used in a closure that returns `Result` or `Option` (or another type that implements `std::ops::Try`)
   --> src/main.rs:173:17
    |
99  |       eventloop.run(move |event, _, controlflow| match event {
    |  ___________________-
100 | |         Event::WindowEvent {
101 | |             event: WindowEvent::CloseRequested,
102 | |             ..
...   |
173 | |                 m.update_instancebuffer(&aetna.allocator)?;
    | |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot use the `?` operator in a closure that returns `()`
...   |
216 | |         _ => {}
217 | |     });
    | |_____- this function should return `Result` or `Option` to accept `?`
    |
    = help: the trait `std::ops::Try` is not implemented for `()`
    = note: required by `std::ops::Try::from_error`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: could not compile `ashen-aetna`.
```
Ooops. Not in the closure. I knew that. (Should have checked where I was correcting the code.) Well, `.expect()` it is, then.

The last one is 
```
warning: unused `std::result::Result` that must be used
   --> src/camera.rs:124:9
    |
124 |         buffer.fill(allocator, &data);
    |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
pointing to 
```rust
    pub fn update_buffer(&self, allocator: &vk_mem::Allocator, buffer: &mut Buffer) {
        let data: [[[f32; 4]; 4]; 2] = [self.viewmatrix.into(), self.projectionmatrix.into()];
        buffer.fill(allocator, &data);
    }
```
of `Camera`. Apparently, this function should have a `Result` as return type.
```rust
    pub fn update_buffer(
        &self,
        allocator: &vk_mem::Allocator,
        buffer: &mut Buffer,
    ) -> Result<(), vk_mem::error::Error> {
        let data: [[[f32; 4]; 4]; 2] = [self.viewmatrix.into(), self.projectionmatrix.into()];
        buffer.fill(allocator, &data)?;
        Ok(())
    }
```
which makes it necessary to handle the same problem at a different place. 

And then, finally, all warnings are dealt with. Now the only output of the program are the validation layer's messages. Many of them are not so
interesting (the program is not producing warnings and errors right now); I think I'll comment out the line 
```rust
//                    | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
```
until I have the feeling that I have to figure out something. Much better if the warnings are easier to spot.


Very well. I think I'm satisfied with the cleanup for now. I'll include the final files. That should make this chapter the worst with respect to "interesting
content per line" (although chapters 000 and 001 may be close ...) 


##### main.rs
```rust
use crate::buffer::Buffer;
use crate::model::{InstanceData, Model};
use crate::pools_and_commandbuffers::{create_commandbuffers, Pools};
use crate::renderpass_and_pipeline::{init_renderpass, Pipeline};
use aetna::Aetna;
use ash::version::DeviceV1_0;
use ash::vk;
use camera::Camera;
use debug::DebugDongXi;
use instance_device_queues::{
    init_device_and_queues, init_instance, init_physical_device_and_properties, QueueFamilies,
    Queues,
};
use nalgebra as na;
use surface::SurfaceDongXi;
use swapchain::SwapchainDongXi;
mod aetna;
mod buffer;
mod camera;
mod debug;
mod instance_device_queues;
mod model;
mod pools_and_commandbuffers;
mod renderpass_and_pipeline;
mod surface;
mod swapchain;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;
    let mut aetna = Aetna::init(window)?;
    let mut cube = Model::cube();
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.0, 0.1))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [0.2, 0.4, 1.0],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.05, 0.05, 0.0))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [1.0, 1.0, 0.2],
    });
    for i in 0..10 {
        for j in 0..10 {
            cube.insert_visibly(InstanceData {
                modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(
                    i as f32 * 0.2 - 1.0,
                    j as f32 * 0.2 - 1.0,
                    0.5,
                )) * na::Matrix4::new_scaling(0.03))
                .into(),
                colour: [1.0, i as f32 * 0.07, j as f32 * 0.07],
            });
            cube.insert_visibly(InstanceData {
                modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(
                    i as f32 * 0.2 - 1.0,
                    0.0,
                    j as f32 * 0.2 - 1.0,
                )) * na::Matrix4::new_scaling(0.02))
                .into(),
                colour: [i as f32 * 0.07, j as f32 * 0.07, 1.0],
            });
        }
    }
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::from_scaled_axis(na::Vector3::new(0.0, 0.0, 1.4))
            * na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0))
            * na::Matrix4::new_scaling(0.1))
        .into(),
        colour: [0.0, 0.5, 0.0],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.5, 0.0, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.5, 0.01, 0.01)))
        .into(),
        colour: [1.0, 0.5, 0.5],
    });
    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.01, 0.5, 0.01)))
        .into(),
        colour: [0.5, 1.0, 0.5],
    });

    cube.insert_visibly(InstanceData {
        modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.0, 0.0))
            * na::Matrix4::new_nonuniform_scaling(&na::Vector3::new(0.01, 0.01, 0.5)))
        .into(),
        colour: [0.5, 0.5, 1.0],
    });
    cube.update_vertexbuffer(&aetna.allocator)?;
    cube.update_indexbuffer(&aetna.allocator)?;
    cube.update_instancebuffer(&aetna.allocator)?;
    aetna.models = vec![cube];
    let mut camera = Camera::builder().build();
    use winit::event::{Event, WindowEvent};
    eventloop.run(move |event, _, controlflow| match event {
        Event::WindowEvent {
            event: WindowEvent::CloseRequested,
            ..
        } => {
            *controlflow = winit::event_loop::ControlFlow::Exit;
        }
        Event::WindowEvent {
            event: WindowEvent::KeyboardInput { input, .. },
            ..
        } => {
            if let winit::event::KeyboardInput {
                state: winit::event::ElementState::Pressed,
                virtual_keycode: Some(keycode),
                ..
            } = input
            {
                match keycode {
                    winit::event::VirtualKeyCode::Right => {
                        camera.turn_right(0.1);
                    }
                    winit::event::VirtualKeyCode::Left => {
                        camera.turn_left(0.1);
                    }
                    winit::event::VirtualKeyCode::Up => {
                        camera.move_forward(0.05);
                    }
                    winit::event::VirtualKeyCode::Down => {
                        camera.move_backward(0.05);
                    }
                    winit::event::VirtualKeyCode::PageUp => {
                        camera.turn_up(0.02);
                    }
                    winit::event::VirtualKeyCode::PageDown => {
                        camera.turn_down(0.02);
                    }
                    _ => {}
                }
            }
        }
        Event::MainEventsCleared => {
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
            camera
                .update_buffer(&aetna.allocator, &mut aetna.uniformbuffer)
                .expect("updating camera buffer");
            for m in &mut aetna.models {
                m.update_instancebuffer(&aetna.allocator)
                    .expect("updating instance buffer impossible");
            }
            aetna
                .update_commandbuffer(image_index as usize)
                .expect("updating the command buffer");
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
```

##### aetna.rs
```rust
use crate::*;
use ash::version::DeviceV1_0;
use ash::version::InstanceV1_0;
use ash::vk;

pub struct Aetna {
    pub window: winit::window::Window,
    _entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    _physical_device: vk::PhysicalDevice,
    _physical_device_properties: vk::PhysicalDeviceProperties,
    _physical_device_features: vk::PhysicalDeviceFeatures,
    pub queue_families: QueueFamilies,
    pub queues: Queues,
    pub device: ash::Device,
    pub swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    pub commandbuffers: Vec<vk::CommandBuffer>,
    pub allocator: vk_mem::Allocator,
    pub models: Vec<Model<[f32; 3], InstanceData>>,
    pub uniformbuffer: Buffer,
    descriptor_pool: vk::DescriptorPool,
    descriptor_sets: Vec<vk::DescriptorSet>,
}

impl Aetna {
    pub fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties, physical_device_features) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;

        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let allocator = vk_mem::Allocator::new(&allocator_create_info)?;

        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &allocator,
        )?;
        let renderpass = init_renderpass(&logical_device, swapchain.surface_format.format)?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;

        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;

        let mut uniformbuffer = Buffer::new(
            &allocator,
            64,
            vk::BufferUsageFlags::UNIFORM_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        let cameratransforms: [[[f32; 4]; 4]; 2] = [
            na::Matrix4::identity().into(),
            na::Matrix4::identity().into(),
        ];
        uniformbuffer.fill(&allocator, &cameratransforms)?;
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::UNIFORM_BUFFER,
            descriptor_count: swapchain.amount_of_images,
        }];
        let descriptor_pool_info = vk::DescriptorPoolCreateInfo::builder()
            .max_sets(swapchain.amount_of_images)
            .pool_sizes(&pool_sizes);
        let descriptor_pool =
            unsafe { logical_device.create_descriptor_pool(&descriptor_pool_info, None) }?;

        let desc_layouts =
            vec![pipeline.descriptor_set_layouts[0]; swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts);
        let descriptor_sets =
            unsafe { logical_device.allocate_descriptor_sets(&descriptor_set_allocate_info) }?;

        for descset in &descriptor_sets {
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: uniformbuffer.buffer,
                offset: 0,
                range: 128,
            }];
            let desc_sets_write = [vk::WriteDescriptorSet::builder()
                .dst_set(*descset)
                .dst_binding(0)
                .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
                .buffer_info(&buffer_infos)
                .build()];
            unsafe { logical_device.update_descriptor_sets(&desc_sets_write, &[]) };
        }

        Ok(Aetna {
            window,
            _entry: entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            _physical_device: physical_device,
            _physical_device_properties: physical_device_properties,
            _physical_device_features: physical_device_features,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            models: vec![],
            uniformbuffer,
            descriptor_pool,
            descriptor_sets,
        })
    }
    pub fn update_commandbuffer(&mut self, index: usize) -> Result<(), vk::Result> {
        let commandbuffer = self.commandbuffers[index];
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            self.device
                .begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [
            vk::ClearValue {
                color: vk::ClearColorValue {
                    float32: [0.0, 0.0, 0.08, 1.0],
                },
            },
            vk::ClearValue {
                depth_stencil: vk::ClearDepthStencilValue {
                    depth: 1.0,
                    stencil: 0,
                },
            },
        ];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(self.renderpass)
            .framebuffer(self.swapchain.framebuffers[index])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: self.swapchain.extent,
            })
            .clear_values(&clearvalues);
        unsafe {
            self.device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            self.device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.pipeline,
            );
            self.device.cmd_bind_descriptor_sets(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.layout,
                0,
                &[self.descriptor_sets[index]],
                &[],
            );
            for m in &self.models {
                m.draw(&self.device, commandbuffer);
            }
            self.device.cmd_end_render_pass(commandbuffer);
            self.device.end_command_buffer(commandbuffer)?;
        }
        Ok(())
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.device
                .destroy_descriptor_pool(self.descriptor_pool, None);
            self.allocator
                .destroy_buffer(self.uniformbuffer.buffer, &self.uniformbuffer.allocation)
                .expect("buffer destruction");
            for m in &self.models {
                if let Some(vb) = &m.vertexbuffer {
                    self.allocator
                        .destroy_buffer(vb.buffer, &vb.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.instancebuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
                if let Some(ib) = &m.indexbuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
            }
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device, &self.allocator);
            self.allocator.destroy();
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
```

##### buffers.rs
```rust
use ash::vk;
pub struct Buffer {
    pub buffer: vk::Buffer,
    pub allocation: vk_mem::Allocation,
    _allocation_info: vk_mem::AllocationInfo,
    pub size_in_bytes: u64,
    buffer_usage: vk::BufferUsageFlags,
    memory_usage: vk_mem::MemoryUsage,
}

impl Buffer {
    pub fn new(
        allocator: &vk_mem::Allocator,
        size_in_bytes: u64,
        buffer_usage: vk::BufferUsageFlags,
        memory_usage: vk_mem::MemoryUsage,
    ) -> Result<Buffer, vk_mem::error::Error> {
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: memory_usage,
            ..Default::default()
        };
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(size_in_bytes)
                .usage(buffer_usage)
                .build(),
            &allocation_create_info,
        )?;
        Ok(Buffer {
            buffer,
            allocation,
            _allocation_info: allocation_info,
            size_in_bytes,
            buffer_usage,
            memory_usage,
        })
    }
    pub fn fill<T: Sized>(
        &mut self,
        allocator: &vk_mem::Allocator,
        data: &[T],
    ) -> Result<(), vk_mem::error::Error> {
        let bytes_to_write = (data.len() * std::mem::size_of::<T>()) as u64;
        if bytes_to_write > self.size_in_bytes {
            allocator.destroy_buffer(self.buffer, &self.allocation)?;
            let newbuffer = Buffer::new(
                allocator,
                bytes_to_write,
                self.buffer_usage,
                self.memory_usage,
            )?;
            *self = newbuffer;
        }
        let data_ptr = allocator.map_memory(&self.allocation)? as *mut T;
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), data.len()) };
        allocator.unmap_memory(&self.allocation)?;
        Ok(())
    }
}
```


##### camera.rs
```rust
use crate::Buffer;
use nalgebra as na;
pub struct Camera {
    viewmatrix: na::Matrix4<f32>,
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
    projectionmatrix: na::Matrix4<f32>,
}
pub struct CameraBuilder {
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
}
#[allow(dead_code)]
impl CameraBuilder {
    pub fn build(self) -> Camera {
        if self.far < self.near {
            println!(
                "far plane (at {}) closer than near plane (at {}) — is that right?",
                self.far, self.near
            );
        }
        let mut cam = Camera {
            position: self.position,
            view_direction: self.view_direction,
            down_direction: na::Unit::new_normalize(
                self.down_direction.as_ref()
                    - self
                        .down_direction
                        .as_ref()
                        .dot(self.view_direction.as_ref())
                        * self.view_direction.as_ref(),
            ),
            fovy: self.fovy,
            aspect: self.aspect,
            near: self.near,
            far: self.far,
            viewmatrix: na::Matrix4::identity(),
            projectionmatrix: na::Matrix4::identity(),
        };
        cam.update_projectionmatrix();
        cam.update_viewmatrix();
        cam
    }
    pub fn position(mut self, pos: na::Vector3<f32>) -> CameraBuilder {
        self.position = pos;
        self
    }
    pub fn fovy(mut self, fovy: f32) -> CameraBuilder {
        self.fovy = fovy.max(0.01).min(std::f32::consts::PI - 0.01);
        self
    }
    pub fn aspect(mut self, aspect: f32) -> CameraBuilder {
        self.aspect = aspect;
        self
    }
    pub fn near(mut self, near: f32) -> CameraBuilder {
        if near <= 0.0 {
            println!("setting near plane to negative value: {} — you sure?", near);
        }
        self.near = near;
        self
    }
    pub fn far(mut self, far: f32) -> CameraBuilder {
        if far <= 0.0 {
            println!("setting far plane to negative value: {} — you sure?", far);
        }
        self.far = far;
        self
    }
    pub fn view_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.view_direction = na::Unit::new_normalize(direction);
        self
    }
    pub fn down_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.down_direction = na::Unit::new_normalize(direction);
        self
    }
}
impl Camera {
    pub fn builder() -> CameraBuilder {
        CameraBuilder {
            position: na::Vector3::new(0.0, -3.0, -3.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, -1.0)),
            fovy: std::f32::consts::FRAC_PI_3,
            aspect: 800.0 / 600.0,
            near: 0.1,
            far: 100.0,
        }
    }
    fn update_projectionmatrix(&mut self) {
        let d = 1.0 / (0.5 * self.fovy).tan();
        self.projectionmatrix = na::Matrix4::new(
            d / self.aspect,
            0.0,
            0.0,
            0.0,
            0.0,
            d,
            0.0,
            0.0,
            0.0,
            0.0,
            self.far / (self.far - self.near),
            -self.near * self.far / (self.far - self.near),
            0.0,
            0.0,
            1.0,
            0.0,
        );
    }
    pub fn update_buffer(
        &self,
        allocator: &vk_mem::Allocator,
        buffer: &mut Buffer,
    ) -> Result<(), vk_mem::error::Error> {
        let data: [[[f32; 4]; 4]; 2] = [self.viewmatrix.into(), self.projectionmatrix.into()];
        buffer.fill(allocator, &data)?;
        Ok(())
    }
    fn update_viewmatrix(&mut self) {
        let right = na::Unit::new_normalize(self.down_direction.cross(&self.view_direction));
        let m = na::Matrix4::new(
            right.x,
            right.y,
            right.z,
            -right.dot(&self.position), //
            self.down_direction.x,
            self.down_direction.y,
            self.down_direction.z,
            -self.down_direction.dot(&self.position), //
            self.view_direction.x,
            self.view_direction.y,
            self.view_direction.z,
            -self.view_direction.dot(&self.position), //
            0.0,
            0.0,
            0.0,
            1.0,
        );
        self.viewmatrix = self.projectionmatrix * m;
    }
    pub fn move_forward(&mut self, distance: f32) {
        self.position += distance * self.view_direction.as_ref();
        self.update_viewmatrix();
    }
    pub fn move_backward(&mut self, distance: f32) {
        self.move_forward(-distance);
    }
    pub fn turn_right(&mut self, angle: f32) {
        let rotation = na::Rotation3::from_axis_angle(&self.down_direction, angle);
        self.view_direction = rotation * self.view_direction;
        self.update_viewmatrix();
    }
    pub fn turn_left(&mut self, angle: f32) {
        self.turn_right(-angle);
    }
    pub fn turn_up(&mut self, angle: f32) {
        let right = na::Unit::new_normalize(self.down_direction.cross(&self.view_direction));
        let rotation = na::Rotation3::from_axis_angle(&right, angle);
        self.view_direction = rotation * self.view_direction;
        self.down_direction = rotation * self.down_direction;
        self.update_viewmatrix();
    }
    pub fn turn_down(&mut self, angle: f32) {
        self.turn_up(-angle);
    }
}
```

##### debug.rs
```rust
use ash::vk;
pub struct DebugDongXi {
    loader: ash::extensions::ext::DebugUtils,
    messenger: vk::DebugUtilsMessengerEXT,
}
impl DebugDongXi {
    pub fn init(entry: &ash::Entry, instance: &ash::Instance) -> Result<DebugDongXi, vk::Result> {
        let debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
            .message_severity(
                vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
                    | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
//                    | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
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

pub unsafe extern "system" fn vulkan_debug_utils_callback(
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
##### instance_device_queues.rs
```rust
use crate::surface::SurfaceDongXi;
use ash::version::DeviceV1_0;
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;
use ash::vk;
pub fn init_instance(
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
        .pfn_user_callback(Some(crate::debug::vulkan_debug_utils_callback));

    let instance_create_info = vk::InstanceCreateInfo::builder()
        .push_next(&mut debugcreateinfo)
        .application_info(&app_info)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_extension_names(&extension_name_pointers);
    unsafe { entry.create_instance(&instance_create_info, None) }
}

pub fn init_physical_device_and_properties(
    instance: &ash::Instance,
) -> Result<
    (
        vk::PhysicalDevice,
        vk::PhysicalDeviceProperties,
        vk::PhysicalDeviceFeatures,
    ),
    vk::Result,
> {
    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    let mut chosen = None;
    for p in phys_devs {
        let properties = unsafe { instance.get_physical_device_properties(p) };
        let features = unsafe { instance.get_physical_device_features(p) };
        if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
            chosen = Some((p, properties, features));
        }
    }
    Ok(chosen.unwrap())
}

pub struct QueueFamilies {
    pub graphics_q_index: Option<u32>,
    pub transfer_q_index: Option<u32>,
}
impl QueueFamilies {
    pub fn init(
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

pub struct Queues {
    pub graphics_queue: vk::Queue,
    pub transfer_queue: vk::Queue,
}

pub fn init_device_and_queues(
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
    let features = vk::PhysicalDeviceFeatures::builder().fill_mode_non_solid(true);
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_features(&features);
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
```
##### model.rs
```rust
use crate::buffer::Buffer;
use ash::version::DeviceV1_0;
use ash::vk;

#[derive(Debug, Clone)]
pub struct InvalidHandle;
impl std::fmt::Display for InvalidHandle {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "invalid handle")
    }
}
impl std::error::Error for InvalidHandle {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        None
    }
}
pub struct Model<V, I> {
    vertexdata: Vec<V>,
    indexdata: Vec<u32>,
    handle_to_index: std::collections::HashMap<usize, usize>,
    handles: Vec<usize>,
    instances: Vec<I>,
    first_invisible: usize,
    next_handle: usize,
    pub vertexbuffer: Option<Buffer>,
    pub indexbuffer: Option<Buffer>,
    pub instancebuffer: Option<Buffer>,
}
#[allow(dead_code)]
impl<V, I> Model<V, I> {
    pub fn get(&self, handle: usize) -> Option<&I> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            self.instances.get(index)
        } else {
            None
        }
    }
    pub fn get_mut(&mut self, handle: usize) -> Option<&mut I> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            self.instances.get_mut(index)
        } else {
            None
        }
    }
    pub fn is_visible(&self, handle: usize) -> Result<bool, InvalidHandle> {
        if let Some(index) = self.handle_to_index.get(&handle) {
            Ok(index < &self.first_invisible)
        } else {
            Err(InvalidHandle)
        }
    }
    pub fn make_visible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
        //if already visible: do nothing
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index < self.first_invisible {
                return Ok(());
            }
            //else: move to position first_invisible and increase value of first_invisible
            self.swap_by_index(index, self.first_invisible);
            self.first_invisible += 1;
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    pub fn make_invisible(&mut self, handle: usize) -> Result<(), InvalidHandle> {
        //if already invisible: do nothing
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index >= self.first_invisible {
                return Ok(());
            }
            //else: move to position before first_invisible and decrease value of first_invisible
            self.swap_by_index(index, self.first_invisible - 1);
            self.first_invisible -= 1;
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    pub fn insert(&mut self, element: I) -> usize {
        let handle = self.next_handle;
        self.next_handle += 1;
        let index = self.instances.len();
        self.instances.push(element);
        self.handles.push(handle);
        self.handle_to_index.insert(handle, index);
        handle
    }
    pub fn insert_visibly(&mut self, element: I) -> usize {
        let new_handle = self.insert(element);
        self.make_visible(new_handle).ok();
        new_handle
    }
    pub fn remove(&mut self, handle: usize) -> Result<I, InvalidHandle> {
        if let Some(&index) = self.handle_to_index.get(&handle) {
            if index < self.first_invisible {
                self.swap_by_index(index, self.first_invisible - 1);
                self.first_invisible -= 1;
            }
            self.swap_by_index(self.first_invisible, self.instances.len() - 1);
            self.handles.pop();
            self.handle_to_index.remove(&handle);
            //must be Some(), otherwise we couldn't have found an index
            Ok(self.instances.pop().unwrap())
        } else {
            Err(InvalidHandle)
        }
    }
    pub fn swap_by_handle(&mut self, handle1: usize, handle2: usize) -> Result<(), InvalidHandle> {
        if handle1 == handle2 {
            return Ok(());
        }
        if let (Some(&index1), Some(&index2)) = (
            self.handle_to_index.get(&handle1),
            self.handle_to_index.get(&handle2),
        ) {
            self.handles.swap(index1, index2);
            self.instances.swap(index1, index2);
            self.handle_to_index.insert(index1, handle2);
            self.handle_to_index.insert(index2, handle1);
            Ok(())
        } else {
            Err(InvalidHandle)
        }
    }
    fn swap_by_index(&mut self, index1: usize, index2: usize) {
        if index1 == index2 {
            return;
        }
        let handle1 = self.handles[index1];
        let handle2 = self.handles[index2];
        self.handles.swap(index1, index2);
        self.instances.swap(index1, index2);
        self.handle_to_index.insert(index1, handle2);
        self.handle_to_index.insert(index2, handle1);
    }
    pub fn update_vertexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.vertexbuffer {
            buffer.fill(allocator, &self.vertexdata)?;
            Ok(())
        } else {
            let bytes = (self.vertexdata.len() * std::mem::size_of::<V>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::VERTEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.vertexdata)?;
            self.vertexbuffer = Some(buffer);
            Ok(())
        }
    }
    pub fn update_indexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.indexbuffer {
            buffer.fill(allocator, &self.indexdata)?;
            Ok(())
        } else {
            let bytes = (self.indexdata.len() * std::mem::size_of::<u32>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::INDEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.indexdata)?;
            self.indexbuffer = Some(buffer);
            Ok(())
        }
    }
    pub fn update_instancebuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.instancebuffer {
            buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
            Ok(())
        } else {
            let bytes = (self.first_invisible * std::mem::size_of::<I>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::VERTEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.instances[0..self.first_invisible])?;
            self.instancebuffer = Some(buffer);
            Ok(())
        }
    }
    pub fn draw(&self, logical_device: &ash::Device, commandbuffer: vk::CommandBuffer) {
        if let Some(vertexbuffer) = &self.vertexbuffer {
            if let Some(indexbuffer) = &self.indexbuffer {
                if let Some(instancebuffer) = &self.instancebuffer {
                    if self.first_invisible > 0 {
                        unsafe {
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                0,
                                &[vertexbuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                1,
                                &[instancebuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_index_buffer(
                                commandbuffer,
                                indexbuffer.buffer,
                                0,
                                vk::IndexType::UINT32,
                            );
                            logical_device.cmd_draw_indexed(
                                commandbuffer,
                                self.indexdata.len() as u32,
                                self.first_invisible as u32,
                                0,
                                0,
                                0,
                            );
                        }
                    }
                }
            }
        }
    }
}
impl Model<[f32; 3], InstanceData> {
    pub fn cube() -> Model<[f32; 3], InstanceData> {
        let lbf = [-1.0, 1.0, -1.0]; //lbf: left-bottom-front
        let lbb = [-1.0, 1.0, 1.0];
        let ltf = [-1.0, -1.0, -1.0];
        let ltb = [-1.0, -1.0, 1.0];
        let rbf = [1.0, 1.0, -1.0];
        let rbb = [1.0, 1.0, 1.0];
        let rtf = [1.0, -1.0, -1.0];
        let rtb = [1.0, -1.0, 1.0];
        Model {
            vertexdata: vec![lbf, lbb, ltf, ltb, rbf, rbb, rtf, rtb],
            indexdata: vec![
                0, 1, 5, 0, 5, 4, //bottom
                2, 7, 3, 2, 6, 7, //top
                0, 6, 2, 0, 4, 6, //front
                1, 3, 7, 1, 7, 5, //back
                0, 2, 1, 1, 2, 3, //left
                4, 5, 6, 5, 7, 6, //right
            ],
            handle_to_index: std::collections::HashMap::new(),
            handles: Vec::new(),
            instances: Vec::new(),
            first_invisible: 0,
            next_handle: 0,
            vertexbuffer: None,
            indexbuffer: None,
            instancebuffer: None,
        }
    }
}

#[repr(C)]
pub struct InstanceData {
    pub modelmatrix: [[f32; 4]; 4],
    pub colour: [f32; 3],
}
```
##### pools_and_commandbuffer.rs
```rust
use crate::instance_device_queues::QueueFamilies;
use ash::version::DeviceV1_0;
use ash::vk;
pub struct Pools {
    commandpool_graphics: vk::CommandPool,
    commandpool_transfer: vk::CommandPool,
}

impl Pools {
    pub fn init(
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
    pub fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            logical_device.destroy_command_pool(self.commandpool_graphics, None);
            logical_device.destroy_command_pool(self.commandpool_transfer, None);
        }
    }
}

pub fn create_commandbuffers(
    logical_device: &ash::Device,
    pools: &Pools,
    amount: u32,
) -> Result<Vec<vk::CommandBuffer>, vk::Result> {
    let commandbuf_allocate_info = vk::CommandBufferAllocateInfo::builder()
        .command_pool(pools.commandpool_graphics)
        .command_buffer_count(amount);
    unsafe { logical_device.allocate_command_buffers(&commandbuf_allocate_info) }
}
```
##### renderpass_and_pipeline.rs
```rust
use crate::swapchain::SwapchainDongXi;
use ash::version::DeviceV1_0;
use ash::vk;
pub fn init_renderpass(
    logical_device: &ash::Device,
    format: vk::Format,
) -> Result<vk::RenderPass, vk::Result> {
    let attachments = [
        vk::AttachmentDescription::builder()
            .format(format)
            .load_op(vk::AttachmentLoadOp::CLEAR)
            .store_op(vk::AttachmentStoreOp::STORE)
            .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
            .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
            .samples(vk::SampleCountFlags::TYPE_1)
            .build(),
        vk::AttachmentDescription::builder()
            .format(vk::Format::D32_SFLOAT)
            .load_op(vk::AttachmentLoadOp::CLEAR)
            .store_op(vk::AttachmentStoreOp::DONT_CARE)
            .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
            .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
            .samples(vk::SampleCountFlags::TYPE_1)
            .build(),
    ];
    let color_attachment_references = [vk::AttachmentReference {
        attachment: 0,
        layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
    }];
    let depth_attachment_reference = vk::AttachmentReference {
        attachment: 1,
        layout: vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
    };
    let subpasses = [vk::SubpassDescription::builder()
        .color_attachments(&color_attachment_references)
        .depth_stencil_attachment(&depth_attachment_reference)
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

pub struct Pipeline {
    pub pipeline: vk::Pipeline,
    pub layout: vk::PipelineLayout,
    pub descriptor_set_layouts: Vec<vk::DescriptorSetLayout>,
}

impl Pipeline {
    pub fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            for dsl in &self.descriptor_set_layouts {
                logical_device.destroy_descriptor_set_layout(*dsl, None);
            }
            logical_device.destroy_pipeline(self.pipeline, None);
            logical_device.destroy_pipeline_layout(self.layout, None);
        }
    }

    pub fn init(
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
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 1,
                offset: 0,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 2,
                offset: 16,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 3,
                offset: 32,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 4,
                offset: 48,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 5,
                offset: 64,
                format: vk::Format::R32G32B32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 12,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 76,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
            .vertex_attribute_descriptions(&vertex_attrib_descs)
            .vertex_binding_descriptions(&vertex_binding_descs);
        let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::TRIANGLE_LIST);
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
        let depth_stencil_info = vk::PipelineDepthStencilStateCreateInfo::builder()
            .depth_test_enable(true)
            .depth_write_enable(true)
            .depth_compare_op(vk::CompareOp::LESS_OR_EQUAL);
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
        let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
            .descriptor_count(1)
            .stage_flags(vk::ShaderStageFlags::VERTEX)
            .build()];
        let descriptorset_layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs);
        let descriptorsetlayout = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info, None)
        }?;
        let desclayouts = vec![descriptorsetlayout];
        let pipelinelayout_info = vk::PipelineLayoutCreateInfo::builder().set_layouts(&desclayouts);
        let pipelinelayout =
            unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
        let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&shader_stages)
            .vertex_input_state(&vertex_input_info)
            .input_assembly_state(&input_assembly_info)
            .viewport_state(&viewport_info)
            .rasterization_state(&rasterizer_info)
            .multisample_state(&multisampler_info)
            .depth_stencil_state(&depth_stencil_info)
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
            descriptor_set_layouts: desclayouts,
        })
    }
}
```
##### surface.rs
```rust
use ash::vk;
pub struct SurfaceDongXi {
    _xlib_surface_loader: ash::extensions::khr::XlibSurface,
    pub surface: vk::SurfaceKHR,
    surface_loader: ash::extensions::khr::Surface,
}

impl SurfaceDongXi {
    pub fn init(
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
            _xlib_surface_loader: xlib_surface_loader,
            surface,
            surface_loader,
        })
    }
    pub fn get_capabilities(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<vk::SurfaceCapabilitiesKHR, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_capabilities(physical_device, self.surface)
        }
    }
    pub fn get_present_modes(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Vec<vk::PresentModeKHR>, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_present_modes(physical_device, self.surface)
        }
    }
    pub fn get_formats(
        &self,
        physical_device: vk::PhysicalDevice,
    ) -> Result<Vec<vk::SurfaceFormatKHR>, vk::Result> {
        unsafe {
            self.surface_loader
                .get_physical_device_surface_formats(physical_device, self.surface)
        }
    }
    pub fn get_physical_device_surface_support(
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
```
##### swapchain.rs
```rust
use crate::surface::SurfaceDongXi;
use crate::QueueFamilies;
use ash::version::DeviceV1_0;
use ash::vk;
pub struct SwapchainDongXi {
    pub swapchain_loader: ash::extensions::khr::Swapchain,
    pub swapchain: vk::SwapchainKHR,
    pub images: Vec<vk::Image>,
    pub imageviews: Vec<vk::ImageView>,
    pub depth_image: vk::Image,
    depth_image_allocation: vk_mem::Allocation,
    _depth_image_allocation_info: vk_mem::AllocationInfo,
    pub depth_imageview: vk::ImageView,
    pub framebuffers: Vec<vk::Framebuffer>,
    pub surface_format: vk::SurfaceFormatKHR,
    pub extent: vk::Extent2D,
    pub image_available: Vec<vk::Semaphore>,
    pub rendering_finished: Vec<vk::Semaphore>,
    pub may_begin_drawing: Vec<vk::Fence>,
    pub amount_of_images: u32,
    pub current_image: usize,
}

impl SwapchainDongXi {
    pub fn init(
        instance: &ash::Instance,
        physical_device: vk::PhysicalDevice,
        logical_device: &ash::Device,
        surfaces: &SurfaceDongXi,
        queue_families: &QueueFamilies,
        allocator: &vk_mem::Allocator,
    ) -> Result<SwapchainDongXi, Box<dyn std::error::Error>> {
        let surface_capabilities = surfaces.get_capabilities(physical_device)?;
        let extent = surface_capabilities.current_extent;
        let _surface_present_modes = surfaces.get_present_modes(physical_device)?;
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
        let extent3d = vk::Extent3D {
            width: extent.width,
            height: extent.height,
            depth: 1,
        };
        let depth_image_info = vk::ImageCreateInfo::builder()
            .image_type(vk::ImageType::TYPE_2D)
            .format(vk::Format::D32_SFLOAT)
            .extent(extent3d)
            .mip_levels(1)
            .array_layers(1)
            .samples(vk::SampleCountFlags::TYPE_1)
            .tiling(vk::ImageTiling::OPTIMAL)
            .usage(vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT)
            .sharing_mode(vk::SharingMode::EXCLUSIVE)
            .queue_family_indices(&queuefamilies);
        let allocation_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::GpuOnly,
            ..Default::default()
        };
        let (depth_image, depth_image_allocation, depth_image_allocation_info) =
            allocator.create_image(&depth_image_info, &allocation_info)?;
        let subresource_range = vk::ImageSubresourceRange::builder()
            .aspect_mask(vk::ImageAspectFlags::DEPTH)
            .base_mip_level(0)
            .level_count(1)
            .base_array_layer(0)
            .layer_count(1);
        let imageview_create_info = vk::ImageViewCreateInfo::builder()
            .image(depth_image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::D32_SFLOAT)
            .subresource_range(*subresource_range);
        let depth_imageview =
            unsafe { logical_device.create_image_view(&imageview_create_info, None) }?;

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
            depth_image,
            depth_image_allocation,
            _depth_image_allocation_info: depth_image_allocation_info,
            depth_imageview,
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
    pub fn create_framebuffers(
        &mut self,
        logical_device: &ash::Device,
        renderpass: vk::RenderPass,
    ) -> Result<(), vk::Result> {
        for iv in &self.imageviews {
            let iview = [*iv, self.depth_imageview];
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
    pub unsafe fn cleanup(&mut self, logical_device: &ash::Device, allocator: &vk_mem::Allocator) {
        logical_device.destroy_image_view(self.depth_imageview, None);
        allocator
            .destroy_image(self.depth_image, &self.depth_image_allocation)
            .expect("depth image destruction");
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
```

##### ../Cargo.toml
```
[package]
name = "ashen-aetna"
version = "0.1.0"
authors = ["Hoj Senna"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
ash = "0.30.0"
winit = "0.22.0"
vk-shader-macros = "0.2.2"
vk-mem = "0.2.2"
nalgebra = "0.18.0"
```

##### ../shaders/shader.vert
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in mat4 model_matrix;
layout (location=5) in vec3 colour;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_Position = ubo.projection_matrix*ubo.view_matrix*model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
}
```

##### ../shaders/shader.frag
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec4 data_from_the_vertexshader;

void main(){
	theColour= data_from_the_vertexshader;
}
```


[Continue](030_Sphere.md)
