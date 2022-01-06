## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Debug Printf

The validation layers (see [Chapter 3](https://hoj-senna.github.io/ashen-aetna/text/003_Validation_layers.html)) are an excellent help. But sometimes
something goes wrong not in the sense of "the program is broken and crashes or hangs completely", but more in the sense of "well, the screen is
showing something, but this is definitely not right". Occasionally, it helps to do some "visual debugging": Use the coordinates (or whatever might be
wrong) and translate it to colour, look whether/where/... the screen looks too green (or whichever colour value indicates the wrongness) and try to
guess the reason. 

But occasionally, it would be nice to just have some `println!("This is the value: {}, is it wrong?",value);` or `dbg!(coordinate[2]);` in the shader
code (and try the reason for the error from that). This is where the Debug Printf feature comes in. 

Let us assume that as a starting point we have the following vertex shader: 
```glsl 
#version 450

layout (location = 0) in vec2 inpos;

void main(){
    gl_Position=vec4(inpos,0.0,1.0);
}
```
and the following fragment shader:
```glsl
#version 450

layout (location = 0) out vec4 outcolour;

void main(){
        outcolour=vec4(0.2,0.2,0.2,1.0);
    }
``` 
which we call with a vertex buffer with content `[0.0, 0.0, 1.0, 1.0, 1.0, 0.0]`. Let us further assume that we want to print the vertex coordinates
in the vertex shader. 

(The shaders are simple on purpose, they are not the point; the necessary setup is. Also, I'm not showing the complete code this time. The main reason
is that I have rewritten a very large part of the renderer in the meantime (in hindsight, I was not entirely happy with some, let's say, architectural
decisions; but yes, indeed, the chapters I had written were useful for me), but I do not feel I have the time to document the rewrite or its result. 
If some code only bears some resemblance to earlier chapters, that may be the reason. On the other hand, the necessary steps are not difficult, the
right places should be easy to identify; and a small isolated chapter is probably better than nothing.) 

Let us include the writing instruction in the shader: 
```glsl
    debugPrintfEXT("Hello World! %f, %f", inpos[0], inpos[1]);
```
The command works similar to C's [`printf`](https://en.wikipedia.org/wiki/Printf_format_string).

According to the corresponding [part](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GLSL_EXT_debug_printf.txt) of the GLSL spec:
```
    This function has a variable number of arguments, and the first argument
    must be a literal string. Other arguments can be of any type and must
    match the type indicated by the order of the format specifiers in the
    string. No type conversions or overload matching rules are applied to
    the arguments.

    Interpretation of the format specifiers is specified by the client API.
    The set of format specifiers is implementation-dependent, but must
    include at least "%d" and "%i" (int), "%u" (uint), and "%f" (float).
```

Okay, back to work. Because: It doesn't work yet. 

Next step: We have to enable this GLSL extension in the shader:
```glsl 
#extension GL_EXT_debug_printf:enable
```
In total that makes our vertex shader be 
```
#version 450
#extension GL_EXT_debug_printf:enable

layout (location = 0) in vec2 inpos;

void main(){
    debugPrintfEXT("Hello World! %f, %f", inpos[0], inpos[1]);
    gl_Position=vec4(inpos,0.0,1.0);
}
```
And with this, the work on the side of the shaders is done. But our program doesn't yet know what to do with this. Nor does Vulkan, apparently:

```
[Debug][error][validation] "Validation Error: [ VUID-VkShaderModuleCreateInfo-pCode-04147 ] Object 0: handle = 0x5594d0188138, type = VK_OBJECT_TYPE_DEVICE; | MessageID =0x3d492883 | vkCreateShaderModule(): The SPIR-V Extension (SPV_KHR_non_semantic_info) was declared, but none of the requirements were met to use it. The Vulkan spec states: If pCode declares any of the SPIR-V extensions listed in the SPIR-V Environment appendix, one of the corresponding requirements must be satisfied (https://vulkan.lunarg.com/doc/view/1.2.170.0~rc2/linux/1.2-extensions/vkspec.html#VUID-VkShaderModuleCreateInfo-pCode-04147)"
```
We are lacking some extension. Well, let's enable it:
```rust 
    let device_extension_name_pointers: Vec<*const i8> = vec![
    ash::extensions::khr::Swapchain::name().as_ptr(),
    ash::vk::KhrShaderNonSemanticInfoFn::name().as_ptr(),
];
```
This is code near the creation of the device, i.e. before the call to 
```rust 
    let logical_device =
        unsafe { instance.create_device(physical_device, &device_create_info, None)? };
```
We were already using the `Swapchain` extension, now we're adding `Khr::ShaderNonSemanticInfo`. Why `InfoFn`, and why `ash::vk::...` and not
`ash::extensions::khr::ShaderNonSemanticInfo`? Well, as far as I can see, this extension is not completely integrated in `ash` in the same way. (To be
fair, there doesn't seem to be real implementation missing, this extension mainly seems to consist of the function providing the name.) 

Also when creating the instance, we want to enable the right extension. So, in 
```rust 
        let mut debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
            .message_severity(
                vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
                    | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
        //            | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
                    | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
            )
            .message_type(
                //                vk::DebugUtilsMessageTypeFlagsEXT::GENERAL|
                vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE | {
                    vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION
                },
            )
            .pfn_user_callback(Some(vulkan_debug_utils_callback));
        let instance = unsafe {
            entry.create_instance(
                &vk::InstanceCreateInfo::builder()
                    .push_next(&mut debugcreateinfo)
                    .push_next(&mut validation_features)
                    .enabled_layer_names(&layers)
                    .enabled_extension_names(&extensions)
                    .application_info(
                        &vk::ApplicationInfo::builder()
                            .api_version(vk::make_api_version(0, 1, 2, 203)),
                    ),
                None,
            )
        }
        .expect("Unable to create Vulkan instance.");
```
we add 
```rust 
        let mut validation_features = vk::ValidationFeaturesEXT::builder()
            .enabled_validation_features(&[vk::ValidationFeatureEnableEXT::DEBUG_PRINTF]);
``` 
and push this onto the `InstanceCreateInfo`: 
```
        let mut validation_features = vk::ValidationFeaturesEXT::builder()
            .enabled_validation_features(&[vk::ValidationFeatureEnableEXT::DEBUG_PRINTF]);
        let mut debugcreateinfo = vk::DebugUtilsMessengerCreateInfoEXT::builder()
            .message_severity(
                vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
                    | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE
            //        | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
                    | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
            )
            .message_type(
                //                vk::DebugUtilsMessageTypeFlagsEXT::GENERAL|
                vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE | {
                    vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION
                },
            )
            .pfn_user_callback(Some(vulkan_debug_utils_callback));
        let instance = unsafe {
            entry.create_instance(
                &vk::InstanceCreateInfo::builder()
                    .push_next(&mut debugcreateinfo)
                    .push_next(&mut validation_features)
                    .enabled_layer_names(&layers)
                    .enabled_extension_names(&extensions)
                    .application_info(
                        &vk::ApplicationInfo::builder()
                            .api_version(vk::make_api_version(0, 1, 2, 203)),
                    ),
                None,
            )
        }
        .expect("Unable to create Vulkan instance.");
```
This still "doesn't work": The messages from Debug Printf are of level `INFO`, which we'll have to uncomment. As this level may lead to some "noise"
in the validation layer output, let's adjust the debug callback function. 
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
    if severity == "info" {
        let msg=message.to_str().expect("An error occurred in Vulkan debug utils callback. What kind of not-String are you handing me?");
        if msg.contains("DEBUG-PRINTF") {
            let msg = msg
                .to_string()
                .replace("Validation Information: [ UNASSIGNED-DEBUG-PRINTF ]", "");
            println!("[Debug][printf] {:?}", msg);
        }
    } else {
        println!("[Debug][{}][{}] {:?}", severity, ty, message);
    }
    vk::FALSE
}
```
(The new part is the handling of the "info" level case.) 


And: 
```
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 0.000000, 0.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 1.000000"
[Debug][printf] " Object 0: handle = 0x5628d3d7e408, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x92394c89 | Hello World! 1.000000, 0.000000"
``` 
etc. 

Good that there are only three vertices to be drawn... I suspect that pure `debugPrintfEXT` calls, especially in the fragment shader, can get messy quickly, if they are not hidden behind rather specific `if`s. 

But hey, we can print text from our shaders and can have our GPU talk to us in yet another way. 

Also: Thanks to [http://anki3d.org/debugprintf-vulkan/](http://anki3d.org/debugprintf-vulkan/) and [https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/master/docs/debug_printf.md](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/master/docs/debug_printf.md).


[Continue]  ( will again take some time, probably)
