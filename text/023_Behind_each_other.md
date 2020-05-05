## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Behind each other: Depth

This time, let us use the following two cubes: 
```rust
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
```
They overlap, and the blue one is a bit further behind. That's exactly how it looks in the picture. Good. Let's try something: 
```rust
        cube.insert_visibly(InstanceData {
            modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.05, 0.05, 0.0))
                * na::Matrix4::new_scaling(0.1))
            .into(),
            colour: [1.0, 1.0, 0.2],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.0, 0.1))
                * na::Matrix4::new_scaling(0.1))
            .into(),
            colour: [0.2, 0.4, 1.0],
        });
```
Now the blue box is drawn in front of the other. But the coordinates still say it should be further back. The coordinates don't matter, the drawing
order does?! Strange. What's going on? 

What happens when we send these vertices to our graphics pipeline? The vertex shader computes actual coordinates, which are then turned into a set of
instructions "this point on the screen has to be painted", and finally the fragment shader does this painting. 

At no point there is a check "is this point hidden behind another point? If so, don't draw" included. We should add such a check. 

We could include something where for each fragment we store its z-coordinate upon drawing, and only paint it if it is not hidden behind previously
drawn objects. This "something" is a depth buffer.

We render into it, we will need it during the render pass. It sounds related to the framebuffer we set up; there were some `Image`s and `ImageView`s
involved: We'll need something similar here. 

With our swapchain, the `Image`s for rendering colour were set up automatically. (We created the swapchain, and afterwards could retrieve these images.)
The depth image we will have to create by ourselves. 

Let us put this with the other swapchain-related stuff: 
```rust
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
```

Inside `SwapChainDongXi::init()`, we create the image first
```rust
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
```
The `image_type` is still a 2D image, the `format` is one including `D` in its name (for "depth" — note: the 32 bit version comes with a performance
cost, 24 bit would be more reasonable in real applications, unless the higher precision is needed); in this function we already have `extent` as
width/height of the swapchain images, here we need width, height, and depth. No mipmapping, only one layer, no multisampling, only one queue family
(`SharingMode::EXCLUSIVE` and `&queuefamilies` (which we had used earlier, it's an array of the graphics queue family only); the memory can live on
GPU only, we won't have to access it.

Some of the options are quite similar to those we used for swapchain creation. 

Afterwards, we create the corresponding `ImageView`: 
```rust
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
```
Important: the format has to match the one above. A change compared to the other `ImageView`s we created: Now we're interested in `DEPTH`, not in `COLOR`.


These changes mean that the function needs slightly different arguments: 
```rust
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
```

and before calling it, we have to create the `Allocator`, so some re-ordering in `Aetna::init()` is in order. 

We also have to include some more lines in `SwapchainDongXi::cleanup()`, there are a new ImageView and a new Image that we have to take care of: 
```rust
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
```

Okay. We have created a depth image. It's time to use it. We should include it in the `FrameBuffer`. 
```rust
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
```
One line has been changed: The imageviews we are using: `let iview = [*iv, self.depth_imageview];`. 

We are rewarded with the following message: 
```
[Debug][error][validation] "vkCreateFramebuffer(): VkFramebufferCreateInfo attachmentCount of 2 does not match attachmentCount of 1 of renderPass (0x16) being used to create Frame
buffer. The Vulkan spec states: attachmentCount must be equal to the attachment count specified in renderPass (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vk
spec.html#VUID-VkFramebufferCreateInfo-attachmentCount-00876)"
```

We use the framebuffers in a render pass, and, of course, the render pass has to be informed that we want to use another attachment.

```rust
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
        .dst_stage_mask(
            vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT
        )
        .dst_access_mask(
            vk::AccessFlags::COLOR_ATTACHMENT_READ | vk::AccessFlags::COLOR_ATTACHMENT_WRITE
        )
        .build()];
    let renderpass_info = vk::RenderPassCreateInfo::builder()
        .attachments(&attachments)
        .subpasses(&subpasses)
        .dependencies(&subpass_dependencies);
    let renderpass = unsafe { logical_device.create_render_pass(&renderpass_info, None)? };
    Ok(renderpass)
}
```
What's changed? 

First, a random bit of cleanup: I have removed an unnecessary argument.

Secondly, there is a description for a second attachment, our depth attachment. Format as before, `CLEAR` on load, `DONT_CARE` for other operations,
including the store operation. (That's different from the colour attachment, but the depth attachment is only used during rendering and not afterwards
(like for being shown on the screen), so we don't have to keep it.) Its `final_layout` should be a dedicated depth attachment format. (As an aside:
This "stencil" often mentioned together with "depth" is another check to decide whether a pixel should remain undrawn, but we're not using it.) 

Thirdly, there is also an attachment reference for the depth attachment. We — again — indicate its layout, and use the next free attachment number.
This attachment reference has to be added to the subpass. (Small difference to the colour attachment: This time it's only one, not an array. We can
have only one depth buffer at a time.)

Renderpass: done. Next problem: 

```
[Debug][error][validation] "Invalid Pipeline CreateInfo State: pDepthStencilState is NULL when rasterization is enabled and subpass uses a depth/stencil attachment. The Vulkan spec states: If the rasterizerDiscardEnable member of pRasterizationState is VK_FALSE, and subpass uses a depth/stencil attachment, pDepthStencilState must be a valid pointer to a valid VkPipelineDepthStencilStateCreateInfo structure (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-VkGraphicsPipelineCreateInfo-rasterizerDiscardEnable-00752)"
```
We have not told our pipeline what to do with depth information. We didn't need to, because we had nothing depth related. Now we have, so we need a
`DepthStencilState`.

So the `pipeline_info` in `Pipeline::init()` gets an additional line: 
```rust
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
```
The new line is that with "depth" in it. Of course, it doesn't tell too much on its own.
```rust
        let depth_stencil_info = vk::PipelineDepthStencilStateCreateInfo::builder()
            .depth_test_enable(true)
            .depth_write_enable(true)
            .depth_compare_op(vk::CompareOp::LESS_OR_EQUAL);
```
Here we enable the depth test (that's what were doing all of this for), and say that a new fragment should be drawn if its depth (z-value) is less or
equal than the one already present at the same location. We also decide that then the depth value should be updated (`depth_write_enable` — it is also
possible to check whether to draw the new fragment and not change the stored depth).

We are rewarded with a new message: 
```
[Debug][error][validation] "In vkCmdBeginRenderPass() the VkRenderPassBeginInfo struct has a clearValueCount of 1 but there must be at least 2 entries in pClearValues array to account for the highest index attachment in renderPass 0x16 that uses VK_ATTACHMENT_LOAD_OP_CLEAR is 2. Note that the pClearValues array is indexed by attachment number so even if some pClearValues entries between 0 and 1 correspond to attachments that aren\'t cleared they will be ignored. The Vulkan spec states: clearValueCount must be greater than the largest attachment index in renderPass that specifies a loadOp (or stencilLoadOp, if the attachment has a depth/stencil format) of VK_ATTACHMENT_LOAD_OP_CLEAR (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-VkRenderPassBeginInfo-clearValueCount-00902)"
```
This is somewhat self-explanatory: We have a second attachment for our render pass, we have already said we want it cleared, but we haven't yet told
our program which value to use for clearing of this second attachment.  

`renderpass_begininfo` in `fill_commandbuffers()` contains some `clearvalues`, and that's what we have to adjust.
```rust
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
```
The second entry is new, and we set depth to the highest possible value `1.0`, and clear stencil to `0` (still not caring about that component).

By now, the yellow box is always drawn in front of the blue one. That was the goal.


[Continue](024_Motion.md)
