## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Validation layers 

Before we take care of finding the graphics card etc., let us make a detour. Let us have a look at validation layers. If something goes wrong in
Vulkan, the application can just crash. Or, for a different kind or severity of "going wrong", there might be just a black window, while everything
*should* be correct, and not a black window. Debugging in the form of "it crashes/doesn't work as intended -> let's randomly change one thing -> hope
it doesn't crash any more and shows the right thing" is probably the least productive way of debugging possible. If only we could get some information
on what's going on on the graphics card. A Vulkan driver itself — by design — does very little error checking. Less work for the driver, faster
application. And: The application (that is, the programmer) has full control and thus has full responsibility. That's it. Is that it? Fortunately, we
are not limited to the Vulkan driver. On top of it, we can insert additional "layers" with additional functionality. The layers we want to use here
are the validation layers. The turn the "error messages" of crashed program or black screen into messages of the form "you have used the following
command incorrectly: ...", which is much more helpful. Our next goal, therefore, is to turn on the validation layers in our program. 

The validation layers need some place where they can put their messages. That means we will have to provide a callback function. A callback function
that can be called from some external C code and that has certain arguments, which is why we will need some "`extern`" keyword and nasty variable types
(like void pointers; joys of foreign function interfaces):

```rust
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

and, as you notice, we'll include "`use ash::vk`" so that we can avoid typing ash so often. The first two arguments of the function are, essentially,
bit masks carrying some information about the kind of debug message. Ash includes a human-readable interpretation in the corresponding`fmt::Debug` 
implementation (and I'm just changing it to lower case so that the terminal messages are more readable). The third argument is a raw pointer (to
something containing the actual message).
Actually, "`*const T`" seems to be something like "`&T`" and when we can get away with it, we will use `&T` where `*const T` is needed (when we assign it to
some value that should be of type `*const T`). Of course, "`*const T`" lacks some guarantees but it directly translates to C, which is why it has its
place in these function interfaces. The last argument is some pointer of unspecified type (that's what void means, in the end) and we don't use it. 

The return value answers the question "should we skip the call to the driver?" 

Now we have prepared a function that we can call, but we still have to create a "debug messenger" that calls it whenever something important happens. 
And for that, we actually have to enable the validation layer, which is something that should happen at the time of instance creation. Had I promised 
that we'll have closer look at the `InstanceCreateInfo` struct? Now is the time.  

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let instance_create_info = vk::InstanceCreateInfo {
        ..Default::default()
    };
    dbg!(&instance_create_info);
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };
    unsafe { instance.destroy_instance(None) };
    Ok(())
}
```

Here I have extracted the InstanceCreateInfo out of the `create_instance` call, but still keep it populated with the default content. I use `dbg!` to
inspect its content; we could as well look at the [documentation](https://docs.rs/ash/0.30.0/ash/vk/struct.InstanceCreateInfo.html) of `ash::vk::InstanceCreateInfo` 
or read the [Vulkan spec](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstanceCreateInfo.html). (If we want to know what the 
entries are good for, we will have to look in the spec anyways. And it's, in general, actually well worth reading.) What do we have? 
```
 [src/main.rs:10] &instance_create_info = InstanceCreateInfo {
    s_type: INSTANCE_CREATE_INFO,
    p_next: 0x0000000000000000,
    flags: ,
    p_application_info: 0x0000000000000000,
    enabled_layer_count: 0,
    pp_enabled_layer_names: 0x0000000000000000,
    enabled_extension_count: 0,
    pp_enabled_extension_names: 0x0000000000000000,
 }
```
`s_type`: This is a field every struct has. It encodes the type. In Rust, this is redundant: we know the type, even the compiler does. 

`p_next`: Another entry every struct has. A result of clever people writing the specification. Idea: Maybe we will later have to extend this struct,
because we have not thought of something now or because it's necessary only in more exotic use cases that aber better part of some extension layer
than of the core specification. Here is place for a pointer to such an extension struct. It will in the overwhelming majority of cases be NULL. (And,
by the way, note that the "p_" in its name already tells us that it's a pointer ...)

`flags`: Another common field: Some options that could be set. A separate type for each struct. The flags in InstanceCreateInfo are of type
vk::InstanceCreateFlags. (And you can guess how those for other structs are called; or you can look it up in the spec.) For very many structs the
description in the specification reads "flags is reserved for future use". 

These were the boring fields that can (almost always) be filled in by "`..Default::default()`". 
In `p_application_info` we can insert some further struct with information on name of the program, API version, etc. 
`pp_enabled_layer_names` and `pp_enabled_extension_names` are the entries that interest us right now: Here we provide a Vec of pointers to the things we
want to switch on. They are accompanied by a number which is just the length of this Vec. 

Let us first provide some ApplicationInfo: 

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let enginename = std::ffi::CString::new("UnknownGameEngine").unwrap();
    let appname = std::ffi::CString::new("The Black Window").unwrap();
    let app_info = vk::ApplicationInfo {
        p_application_name: appname.as_ptr(),
        p_engine_name: enginename.as_ptr(),
        engine_version: vk::make_version(0, 42, 0),
        application_version: vk::make_version(0, 0, 1),
        api_version: vk::make_version(1, 0, 106),
        ..Default::default()
    };
    let instance_create_info = vk::InstanceCreateInfo {
        p_application_info: &app_info,
        ..Default::default()
    };
    dbg!(&instance_create_info);
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };
    unsafe { instance.destroy_instance(None) };
    Ok(())
}
```

The names are provided as `CStrings`, which are, unsurprisingly, the C equivalents of Strings. For Vecs and Vec-like structures (Strings, CStrings) the
right way to pass them into the ApplicationInfo struct is to use `.as_ptr()`. 
Apparently we are using UnknownGameEngine in version 0.42.0 to create our application. As we do not want to use too much space on version numbers,
these 0, 42 and 0 are combined into a single u32 by `vk::make_version`. 

Time to load the validation layers (their name: "`VK_LAYER_KHRONOS_validation`") and to enable the DebugUtils extension:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let enginename = std::ffi::CString::new("UnknownGameEngine").unwrap();
    let appname = std::ffi::CString::new("The Black Window").unwrap();
    let app_info = vk::ApplicationInfo {
        p_application_name: appname.as_ptr(),
        p_engine_name: enginename.as_ptr(),
        engine_version: vk::make_version(0, 42, 0),
        application_version: vk::make_version(0, 0, 1),
        api_version: vk::make_version(1, 0, 106),
        ..Default::default()
    };
    let layer_names: Vec<std::ffi::CString> =
        vec![std::ffi::CString::new("VK_LAYER_KHRONOS_validation").unwrap()];
    let layer_name_pointers: Vec<*const i8> = layer_names
        .iter()
        .map(|layer_name| layer_name.as_ptr())
        .collect();
    let extension_name_pointers: Vec<*const i8> =
        vec![ash::extensions::ext::DebugUtils::name().as_ptr()];

    let instance_create_info = vk::InstanceCreateInfo {
        p_application_info: &app_info,
        pp_enabled_layer_names: layer_name_pointers.as_ptr(),
        enabled_layer_count: layer_name_pointers.len() as u32,
        pp_enabled_extension_names: extension_name_pointers.as_ptr(),
        enabled_extension_count: extension_name_pointers.len() as u32,
        ..Default::default()
    };
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };
    unsafe { instance.destroy_instance(None) };
    Ok(())
}
```

We have indicated the name of the layer we want to enable, we have converted the whole vector containing it to a vector of pointers and have includde
it in the InstanceCreateInfo struct. For the extension, the same. There the name is provided by ash.

In order to use the new functionality, we need a debug messenger that actually passes messages to the callback function we have written earlier. And
in order to create a debug messenger, we need something like the "entry" we created in the first step, only for the DebugUtils instead of for the
Vulkan driver. After instance creation we include: 


```rust
    let debug_utils = ash::extensions::ext::DebugUtils::new(&entry, &instance);
    let debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT {
        message_severity: vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
            | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
            | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
            | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
        message_type: vk::DebugUtilsMessageTypeFlagsEXT::GENERAL
            | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE
            | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION,
        pfn_user_callback: Some(vulkan_debug_utils_callback),
        ..Default::default()
    };
```

```rust
    let utils_messenger =
        unsafe { debug_utils.create_debug_utils_messenger(&debugcreateinfo, None)? };
```

Messenger creation, as announced. The final "None" again something allocation related that we ignore, the configuration of what we don't ignore takes
place in some CreateInfo struct. (The final "EXT" in its name shows that it belongs to some extension.) We specify: the message types that we want to
receive and the message severities. If you receive too many messages, this is the place where you want to remove "VERBOSE", for example; and finally 
we include the user callback function. 

Let's run it! 
And there is some output: 
```
 [Debug][verbose][general] "Added messenger"
 [Debug][info][validation] "OBJ[0x1] : CREATE DebugUtilsMessengerEXT object 0x1"
 [Debug][verbose][general] "Added messenger"
 [Debug][verbose][general] "Added messenger"
 [Debug][info][validation] "OBJ_STAT Destroy Instance obj 0x558c867229e0 (1 total objs remain & 0 Instance objs)."
 [Debug][info][validation] "OBJ_STAT Destroy Instance obj 0x558c867229e0 (1 total objs remain & 0 Instance objs)."
 [Debug][verbose][general | validation] "Destroyed messenger\n"
 [Debug][verbose][general | validation] "Destroyed messenger\n"
 [Debug][error][validation] "Debug messengers not removed before DestroyInstance"
 [Debug][error][validation] "Debug messengers not removed before DestroyInstance"
 [Debug][error][validation] "Debug messengers not removed before DestroyInstance"
 [Debug][error][validation] "Debug messengers not removed before DestroyInstance"
 [Debug][info][general] "Unloading layer library libVkLayer_khronos_validation.so"
```
First observation: Every message (except the first two and the last) is duplicated. I don't know why (some bug? some suboptimal installation?), but there could
be worse. 
Second observation: There is an actual error being reported. We should have removed the debug messenger before destroying the instance. Okay, let's
add this to the final unsafe block: 

```rust
        debug_utils.destroy_debug_utils_messenger(utils_messenger, None);
```

Run it again, the error messages should have disappeared. 
By now, our program looks as follows: 

```rust
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;
use ash::vk;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let enginename = std::ffi::CString::new("UnknownGameEngine").unwrap();
    let appname = std::ffi::CString::new("The Black Window").unwrap();
    let app_info = vk::ApplicationInfo {
        p_application_name: appname.as_ptr(),
        p_engine_name: enginename.as_ptr(),
        engine_version: vk::make_version(0, 42, 0),
        application_version: vk::make_version(0, 0, 1),
        api_version: vk::make_version(1, 0, 106),
        ..Default::default()
    };
    let layer_names: Vec<std::ffi::CString> =
        vec![std::ffi::CString::new("VK_LAYER_KHRONOS_validation").unwrap()];
    let layer_name_pointers: Vec<*const i8> = layer_names
        .iter()
        .map(|layer_name| layer_name.as_ptr())
        .collect();
    let extension_name_pointers: Vec<*const i8> =
        vec![ash::extensions::ext::DebugUtils::name().as_ptr()];

    let instance_create_info = vk::InstanceCreateInfo {
        p_application_info: &app_info,
        pp_enabled_layer_names: layer_name_pointers.as_ptr(),
        enabled_layer_count: layer_name_pointers.len() as u32,
        pp_enabled_extension_names: extension_name_pointers.as_ptr(),
        enabled_extension_count: extension_name_pointers.len() as u32,
        ..Default::default()
    };
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };

    let debug_utils = ash::extensions::ext::DebugUtils::new(&entry, &instance);
    let debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT {
        message_severity: vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
            | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
            | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
            | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
        message_type: vk::DebugUtilsMessageTypeFlagsEXT::GENERAL
            | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE
            | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION,
        pfn_user_callback: Some(vulkan_debug_utils_callback),
        ..Default::default()
    };

    let utils_messenger =
        unsafe { debug_utils.create_debug_utils_messenger(&debugcreateinfo, None)? };

    unsafe {
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

There is something annoying in the creation of the ...Info structs: always those "`..Default::default()`", and redundant information in form of the
lengths of Vecs that we have to provide explicitly. Before continuing, let's make that nicer. Because (as I learned just when writing this) there is a
nicer way to do it: We can use the builders ash includes:

```rust
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

    let instance_create_info = vk::InstanceCreateInfo::builder()
        .application_info(&app_info)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_extension_names(&extension_name_pointers);
    let instance = unsafe { entry.create_instance(&instance_create_info, None)? };

    let debug_utils = ash::extensions::ext::DebugUtils::new(&entry, &instance);
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
    let utils_messenger =
        unsafe { debug_utils.create_debug_utils_messenger(&debugcreateinfo, None)? };
```

We have replaced every `CreateInfo{ ...}` by `CreateInfo::builder()` and have then set all necessary information with a separate function invocation. In
other words every "some_field_name: value" has turned into ".some_field_name(value)". (Note that the annotations for pointers were dropped: it's
`.enabled_layer_names` for `pp_enabled_layer_names`.) And we got rid of the final `..Default::default()`, and of the redundant lengths of Vec: the function 
setting enabled_layer_names takes care of enabled_layer_counts, too. We can also do not need some of the `.as_ptr()` calls. Convenient. 

Important to note if you are somewhat familiar with this "builder pattern" is the following: We do not call a final `.build()`. Instead we directly pass
a reference to the builder where a reference of the struct to be built would have been needed. This is possible because the builder implements the
right Deref trait. Calling build screws with lifetimes of some content of the struct: There are a lot of raw pointers involved and dangling pointers
are ... something to avoid. Do not call `build()` on anything Vulkan-related (unless you know what you're doing — or do not seem to have a better choice
...). 

Are we done with the debugging functionality? We could say "yes", but for the sake of a bit more of completeness, let's observe the following: The
debug messenger is created after instance creation and destroyed before instance destruction. That means that we cannot receive any information if
instance creation or destruction go wrong. This is where the `p_next` field of the InstanceCreateInfo struct shines: Here we can include debugcreateinfo 
(and the DebugUtils extension then knows what to do with it: create a debug messenger that can inform us about problems with instance creation etc.) 
So: 

```rust
    let instance_create_info = vk::InstanceCreateInfo::builder()
        .push_next(&mut debugcreateinfo)
        .application_info(&app_info)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_extension_names(&extension_name_pointers);
```

(and, of course, we have to move let debugcreateinfo to some earlier place and have to insert some "`mut`"). If we wanted to avoid the builder variant
(going for the "direct, more obvious correspondence to the Vulkan spec" feel), we could write 

```rust
            p_next: &debugcreateinfo as *const vk::DebugUtilsMessengerCreateInfoEXT as *const c_void
```

but I think I prefer the `.push_next` variant ... 
If I run the program now, there is a lot of new messages. Okay, now we're done with the debug setup. 

[Continue](004_Physical_device.md)
