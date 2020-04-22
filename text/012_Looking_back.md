## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Looking back from the first program with visual output

It was a long way to the first drawn point on screen. It is quite easy to get lost in the details, there was a lot of setup involved. Let's try to
have a look back — without bothering with the details: What have we done, what were the *important* points? 

We wanted to draw. Somewhere there is literally a "draw" command. Maybe we should start there: 
```rust
            logical_device.cmd_draw(commandbuffer, 1, 1, 0, 0);
```
Two obvious questions spring to mind: 1. How do we get this command to the GPU? 2. What does "drawing" mean? Or more granularly: 2a. What do we want
to draw? 2b. What, besides a description of what to draw, do we need so that it becomes visible on the screen? 

* 1  Commands are recorded in command buffers, these command buffers are submitted to queues.  

* 2b. First of all, there has to *be* a screen, or something to draw on. We made a detour from Vulkan to set up a window with winit, and made it
available to Vulkan as a "surface". Associated to this surface — so as to not having to draw on what is currently shown, but to prepare the picture in
advance and *then* show it — we created a swapchain. This swapchain consisted of several images, needing some "imageviews" (info on how to interpret
the data) and framebuffers to associate them to something to be affected by draw commands (wait for 2a for more). 
Working with the swapchain was "acquire next image", "submit draw commands", "present the image", repeat. As these steps have to occur in the right
order, so there was some synchronisation to deal with.

* 2a. The central part of an answer to this question is the graphics pipeline. Pipelines are used inside of renderpasses (connecting them with
framebuffers, see 2b). A pipeline consists of the program that is executed on the GPU and has different stages, some of which appear to us as,
essentially, a few options to set ("fixed function stages"), some of which are entirely optional (and have not been touched by us), and some of which
we have supplied with separate programs, shader programs, of their own. The absolute minimum are a vertex shader and a fragment shader. The draw
command above calls the vertex shader once. The vertex shader's main responsibility is to turn some data into "these points on the screen should be
painted" and pass this information to the fragment shader, which runs for each of those pixels, and outputs a colour.


What else was there? Of course, some further setup. Before queues and whatever else we needed exist and can be accessed, they have to be created. The
creation begins by finding a Vulkan instance, a physical device, and setting up a logical device (the abstraction that almost all of our later Vulkan
calls will pass through). 

Despite our rather modest goal, we have already had to venture beyond Vulkan's core functionality. Different ways of doing that were a) using
additional "layers" ("between" the application and the driver), and b) using "extensions". 
We came across a) with the validation layers; and b) with debug again and in the context of surfaces (because that's platform specific) and with the
swapchain (although this extension is an extension on the level of a single device — not that we'd be using several of those). Each extension comes
with some loader carrying all related functions as its methods.

[Continue](013_Coordinates.md)
