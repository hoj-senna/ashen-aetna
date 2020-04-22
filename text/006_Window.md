## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# A window

We have created a device, we have something (queues) to which we could submit commands (okay, if we had prepared some commandbuffers; and for most 
commands we'd probably need some additional information; it's difficult to "bind a pipeline" if no pipeline has been created, whatever that may be ...) —
however, there is something that is *obviously* missing. A window, something to draw on. Now, windows are optional. We *could* be interested in Vulkan and the GPU only for its
computing capabilities. Usually, we aren't. This is supposed to be about graphics, after all. Therefore, let's create a window and let's get closer to
drawing in it. Firstly, a window. This is not a task for Vulkan. We will use the winit crate instead. Cargo.toml gets an additional line
`winit="0.22.0"` and we insert the following lines at the beginning of main: 
```rust
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;
```
Now the program opens and closes a window. If we want to render into this window, we have to acquaint the vulkan part of our program with the window
just created, that is, somehow tell it that there is this window "surface" and we want to use it. This is a somewhat closer interaction with the
operating system, meaning that the specifics may differ depending on the operating system you are using. If it's Linux, you have good chances that the
following works; for others you should have a brief look at some examples or other tutorials and adapt the following few paragraphs accordingly. (I
have no way to test other platforms, so I'll stick with the one variant my computer is running.) 
On the Vulkan side, we require another extension to deal with all the surface stuff. On the winit side, we have to extract some of the window
internals. Let us write the following lines between creation of the debug messenger and the code dealing with the physical device: 
```rust
    use winit::platform::unix::WindowExtUnix;
    let x11_display = window.xlib_display().unwrap();
    let x11_window = window.xlib_window().unwrap();
    let x11_create_info = vk::XlibSurfaceCreateInfoKHR::builder()
        .window(x11_window)
        .dpy(x11_display as *mut vk::Display);
    let xlib_surface_loader = ash::extensions::khr::XlibSurface::new(&entry, &instance);
    let surface = unsafe { xlib_surface_loader.create_xlib_surface(&x11_create_info, None) }?;
    let surface_loader = ash::extensions::khr::Surface::new(&entry, &instance);
```
The first two assignments deal with the window internals; they need a certain trait in scope, hence the use clause. We then fill some CreateInfo,
providing window and display, create an `xlib_surface_loader` and get the surface; as well as a surface loader (an entry to surface-related functions).
Running the program right now results in a panic: "`Unable to load create_xlib_surface_khr`" - We should also load these extensions! This means a small
update in some earlier code: 
```rust
    let extension_name_pointers: Vec<*const i8> = vec![
        ash::extensions::ext::DebugUtils::name().as_ptr(),
        ash::extensions::khr::Surface::name().as_ptr(),
        ash::extensions::khr::XlibSurface::name().as_ptr(),
    ];
```
And we should also add 
```rust
        surface_loader.destroy_surface(surface, None);
```
at the end. Okay. Now it would be nice if the queue (family) that we selected could also deal with this surface and draw on it. We will check this
where we search for a graphics queue.
```rust
	if qfam.queue_count > 0 && qfam.queue_flags.contains(vk::QueueFlags::GRAPHICS)
             && unsafe {
                surface_loader
                    .get_physical_device_surface_support(physical_device, index as u32, surface)?
            }
            {
                found_graphics_q_index = Some(index as u32);
            }
```
This comes with the warning that (theoretically) it is possible that the (only) queue available for presenting on the surface and the (only) queues
capable of "GRAPHICS" drawing differ. This is not the case here, and this is tutorial code only, so we'll ignore this potential problem. 

[Continue](007_Swapchain.md)
