## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Resizing
Have you ever tried changing the window size of our program while it was running? 

Don't; I can tell you what happens: 
```
thread 'main' panicked at 'queue presentation: ERROR_OUT_OF_DATE_KHR', src/main.rs:216:17
```
That refers to the following code: 
```rust
            unsafe {
                aetna
                    .swapchain
                    .swapchain_loader
                    .queue_present(aetna.queues.graphics_queue, &present_info)
                    .expect("queue presentation");
            };
```
and the problem is that the swapchain and the surface do not match any more. The `expect` could have told us that something could go wrong here. Let's
handle it slightly better: 
```rust
            unsafe {
                match aetna
                    .swapchain
                    .swapchain_loader
                    .queue_present(aetna.queues.graphics_queue, &present_info)
                {
                    Ok(..) => {}
                    Err(ash::vk::Result::ERROR_OUT_OF_DATE_KHR) => {
                        aetna.recreate_swapchain().expect("swapchain recreation");
                    }
                    _ => {
                        panic!("unhandled queue presentation error");
                    }
                }
            };
```
This means that we'll also need a `recreate_swapchain` function.

Something like this: 
```rust
    pub fn recreate_swapchain(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        unsafe {
            self.swapchain.cleanup(&self.device, &self.allocator);
        }
        self.swapchain = SwapchainDongXi::init(
            &self.instance,
            self.physical_device,
            &self.device,
            &self.surfaces,
            &self.queue_families,
            &self.allocator,
        )?;
        Ok(())
    }
```
(I've taken this opportunity to remane `_physical_device` to `physical_device`.)

What happens now? Well ... 
```
[Debug][error][validation] "Cannot call vkDestroyImageView on VkImageView 0xc000000000c[] that is currently in use by a command buffer. The Vulkan spec states: All submitted comma
nds that refer to imageView must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyImageView-imageView-01026)"
[Debug][error][validation] "Cannot call vkDestroyImage on VkImage 0xa000000000a[] that is currently in use by a command buffer. The Vulkan spec states: All submitted commands that
 refer to image, either directly or via a VkImageView, must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyI
mage-image-01000)"
[Debug][error][validation] "VkFence 0xf000000000f[] is in use. The Vulkan spec states: All queue submission commands that refer to fence must have completed execution (https://www
.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFence-fence-01120)"
[Debug][error][validation] "VkFence 0x120000000012[] is in use. The Vulkan spec states: All queue submission commands that refer to fence must have completed execution (https://ww
w.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFence-fence-01120)"
[Debug][error][validation] "VkFence 0x150000000015[] is in use. The Vulkan spec states: All queue submission commands that refer to fence must have completed execution (https://ww
w.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFence-fence-01120)"
[Debug][error][validation] "Cannot call vkDestroySemaphore on VkSemaphore 0xe000000000e[] that is currently in use by a command buffer. The Vulkan spec states: All submitted batch
es that refer to semaphore must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroySemaphore-semaphore-01137)"
[Debug][error][validation] "Cannot call vkDestroySemaphore on VkSemaphore 0x110000000011[] that is currently in use by a command buffer. The Vulkan spec states: All submitted batc
hes that refer to semaphore must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroySemaphore-semaphore-01137)"
[Debug][error][validation] "Cannot call vkDestroySemaphore on VkSemaphore 0x140000000014[] that is currently in use by a command buffer. The Vulkan spec states: All submitted batc
hes that refer to semaphore must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroySemaphore-semaphore-01137)"
[Debug][error][validation] "Cannot call vkDestroyFramebuffer on VkFramebuffer 0x170000000017[] that is currently in use by a command buffer. The Vulkan spec states: All submitted 
commands that refer to framebuffer must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFramebuffer-framebuff
er-00892)"
[Debug][error][validation] "Cannot call vkDestroyFramebuffer on VkFramebuffer 0x180000000018[] that is currently in use by a command buffer. The Vulkan spec states: All submitted 
commands that refer to framebuffer must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFramebuffer-framebuff
er-00892)"
[Debug][error][validation] "Cannot call vkDestroyFramebuffer on VkFramebuffer 0x190000000019[] that is currently in use by a command buffer. The Vulkan spec states: All submitted 
commands that refer to framebuffer must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyFramebuffer-framebuff
er-00892)"
[Debug][error][validation] "Cannot call vkDestroyImageView on VkImageView 0x70000000007[] that is currently in use by a command buffer. The Vulkan spec states: All submitted comma
nds that refer to imageView must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyImageView-imageView-01026)"
[Debug][error][validation] "Cannot call vkDestroyImageView on VkImageView 0x80000000008[] that is currently in use by a command buffer. The Vulkan spec states: All submitted comma
nds that refer to imageView must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyImageView-imageView-01026)"
[Debug][error][validation] "Cannot call vkDestroyImageView on VkImageView 0x90000000009[] that is currently in use by a command buffer. The Vulkan spec states: All submitted comma
nds that refer to imageView must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkDestroyImageView-imageView-01026)"
[Debug][error][validation] "vkCreateSwapchainKHR() called with imageExtent = (862,600), which is outside the bounds returned by vkGetPhysicalDeviceSurfaceCapabilitiesKHR(): curren
tExtent = (872,600), minImageExtent = (872,600), maxImageExtent = (872,600). The Vulkan spec states: imageExtent must be between minImageExtent and maxImageExtent, inclusive, wher
e minImageExtent and maxImageExtent are members of the VkSurfaceCapabilitiesKHR structure returned by vkGetPhysicalDeviceSurfaceCapabilitiesKHR for the surface (https://www.khrono
s.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-VkSwapchainCreateInfoKHR-imageExtent-01274)"
[Debug][error][validation] "Calling vkBeginCommandBuffer() on active VkCommandBuffer 0x55c181cda8d0[] before it has completed. You must check command buffer fence before this call
. The Vulkan spec states: commandBuffer must not be in the recording or pending state. (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkBeginC
ommandBuffer-commandBuffer-00049)"
thread 'main' panicked at 'index out of bounds: the len is 0 but the index is 0', /rustc/4fb7144ed159f94491249e86d5bbd033b5d60550/src/libcore/slice/mod.rs:2842:10
```
We should better make sure that the things we destroy are not being busy. Ok, let's wait first:
```rust
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
        }
```
Fewer errors: 
```
thread 'main' panicked at 'index out of bounds: the len is 0 but the index is 0',
```
(without the validation errors in front) — The error, by the way, refers to a place in the `update_commandbuffer` function: 
```rust
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(self.renderpass)
            .framebuffer(self.swapchain.framebuffers[index])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: self.swapchain.extent,
            })
            .clear_values(&clearvalues);
```
more precisely: `self.swapchain.framebuffers[index]`, and, indeed, `swapchain`, in our implementation, needs a separate call to `create_framebuffers`:
```rust
        self.swapchain
            .create_framebuffers(&self.device, self.renderpass)?;
```
That takes care of the error. 

However, we should expect the visible image to scale with the window (x=-1 always on the left, x=+1 always on the right) and it doesn't. Also: After
enlargening, some areas of the window do not display the spheres, even if that's where those should be.

That sounds like an issue of viewport and/or scissors, part of the graphics pipeline. Two more lines for the `recreate_swapchain` function:
```rust
        self.pipeline.cleanup(&self.device);
        self.pipeline = Pipeline::init(&self.device, &self.swapchain, &self.renderpass)?;
```
Okay: No more errors. 

But the image looks stretched (if the window was not just scaled uniformly in both directions). We therefore should also update the camera. Its aspect
ratio is what we want to change here. 

Let's introduce a method for `Camera`:
```rust
    pub fn set_aspect(&mut self, aspect: f32) {
        self.aspect = aspect;
        self.update_projectionmatrix();
    }
```
(It's a whole function instead of just a public field, because we want to ensure that also the projection matrix is up to date.)

And then we call this function, not forgetting about updating the buffer either: 
```rust
                    Err(ash::vk::Result::ERROR_OUT_OF_DATE_KHR) => {
                        aetna.recreate_swapchain().expect("swapchain recreation");
                        camera.set_aspect(
                            aetna.swapchain.extent.width as f32
                                / aetna.swapchain.extent.height as f32,
                        );
                        camera
                            .update_buffer(&aetna.allocator, &mut aetna.uniformbuffer)
                            .expect("camera buffer update");
                    }
```

[Continue](037_Texture.md)
