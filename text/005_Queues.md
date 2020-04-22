## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

## Queues

 Queues are where we submit commands to; in some sense they are the place where our graphics commands stand in a queue and wait.
Queue families are collections of such queues, grouped by their "specialization". They may be made for graphics commands, or for transfer operations,
or for computing. Which queue families are available is a property of the physical device (and this might also play a role in selection of said
device). We choose one capable of graphics and one suitable for transfer operations. For transfer we prefer a queue family that is dedicated to
transfer only, hoping that we thereby hit specialised hardware which is faster. But we could also just use the same. Of course, we only consider queue
families that actually contain queues ... 
```rust
    let queuefamilyproperties =
        unsafe { instance.get_physical_device_queue_family_properties(physical_device) };
    dbg!(&queuefamilyproperties);
    let qfamindices = {
        let mut found_graphics_q_index = None;
        let mut found_transfer_q_index = None;
        for (index, qfam) in queuefamilyproperties.iter().enumerate() {
            if qfam.queue_count > 0 && qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
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
```
Having found these queue family indices, we next create a logical device, as primary interface to the physical device. 
```rust
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
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_layer_names(&layer_name_pointers);
    let logical_device =
        unsafe { instance.create_device(physical_device, &device_create_info, None)? };
    let graphics_queue = unsafe { logical_device.get_device_queue(qfamindices.0, 0) };
    let transfer_queue = unsafe { logical_device.get_device_queue(qfamindices.1, 0) };

    unsafe {
        logical_device.destroy_device(None);
        debug_utils.destroy_debug_utils_messenger(utils_messenger, None);
        instance.destroy_instance(None)
    };
    Ok(())
```

Before I forget to mention it: We also destroy this device at the end. 

Let us look at the `DeviceCreateInfo`: "`enabled_layer_names`" is given the information that we use the validation layers. This is not important, the
field will be ignored (by recent implementations). We mainly set it for compatibility reasons. Apart from that, we pass in some information on queues.
We specify which queue family we want to use (those found previously), how many queues we need from that family, and what their priorities should be. 
Priorities range between 0.0 and 1.0, and higher value means higher priority, that is, preferential treatment. The amount of queues is specified, but
not explicitly: If we wanted to have three queues for one of these families, we'd set more priorities: `let priorities = [1.0f32,1.0f32,0.5f32]`, for
example. Also, on the `DeviceCreationInfoBuilder` we use `.build()`, despite my warning not to do that. It seems necessary, because these things are
collected in an array, and Deref does not reach inside, so this seems to be the wrong type. I had warned to not do this, unless one knew what one were
doing. I don't, but something similar happens in the examples in the ash repository, so we should be fine. 

What is not really fine is the case that both `DeviceQueueCreateInfo`s might refer to the same queuefamily (if the indices for graphics and transfer 
queue were the same). Then we should use only one DeviceQueueCreateInfo (and possibly request two queues there). Exercise: deal with that. I don't want to. 

Moving on: Creating the device also creates the queues, but we still have to request them (their handles), so that we can refer to them later. We
provide the index of the queue family and the index of the queue within said family.

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
    let extension_name_pointers: Vec<*const i8> =
        vec![ash::extensions::ext::DebugUtils::name().as_ptr()];
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
            if qfam.queue_count > 0 && qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
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
    dbg!(&queue_infos);
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_layer_names(&layer_name_pointers);
    let logical_device =
        unsafe { instance.create_device(physical_device, &device_create_info, None)? };
    let graphics_queue = unsafe { logical_device.get_device_queue(qfamindices.0, 0) };
    let transfer_queue = unsafe { logical_device.get_device_queue(qfamindices.1, 0) };

    unsafe {
        logical_device.destroy_device(None);
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

[Continue](006_Window.md)
