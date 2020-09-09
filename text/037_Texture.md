## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# A Texture

Up to now, all data we sent to the GPU was, in some sense, the same for the whole model (or, at the very least, there was at most one value per
vertex). But what if we want to vary some value (say, the colour) over one triangle — in a more complicated way than linearly interpolating between
the values at the vertices? What if we want to draw a picture, say from a .jpg file, to the screen?  It would be very impractical to invent a vertex
for every pixel. And if we want to put the picture not on the screen but on the surface of some model? 

There is a way to deal with this "problem": Textures. We load an image, put it on the GPU and when drawing we ask: "What's the (colour) value at this
particular point of the image?" 


I'd like to take a step back from the scene we have been rendering and use a simpler setting. Let us start from the following simple vertex data and model (which we
will further modify during this chapter): 


```rust
#[derive(Copy, Clone, Debug)]
#[repr(C)]
pub struct TexturedVertexData {
    pub position: [f32; 3],
}

#[repr(C)]
pub struct TexturedInstanceData {
    pub modelmatrix: [[f32; 4]; 4],
    pub inverse_modelmatrix: [[f32; 4]; 4],
}

impl TexturedInstanceData {
    pub fn from_matrix(modelmatrix: na::Matrix4<f32>) -> TexturedInstanceData {
        TexturedInstanceData {
            modelmatrix: modelmatrix.into(),
            inverse_modelmatrix: modelmatrix.try_inverse().unwrap().into(),
        }
    }
}

impl Model<TexturedVertexData, TexturedInstanceData> {
    pub fn quad() -> Self {
        let lb = TexturedVertexData {
            position: [-1.0, 1.0, 0.0],
        }; //lb: left-bottom
        let lt = TexturedVertexData {
            position: [-1.0, -1.0, 0.0],
        };
        let rb = TexturedVertexData {
            position: [1.0, 1.0, 0.0],
        };
        let rt = TexturedVertexData {
            position: [1.0, -1.0, 0.0],
        };
        Model {
            vertexdata: vec![lb, lt, rb, rt],
            indexdata: vec![0, 2, 1, 1, 2, 3],
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
```

This means that `Aetna` has to contain `pub models: Vec<Model<TexturedVertexData, TexturedInstanceData>>`.

Shaders (`shader_textured.vert` and `shader_textured.frag`) are also rather simple. In the vertex shader, we keep position data and the ability to move the camera, but we
only pass some fixed colour value to the fragment shader, which does much less work than the previous one with all the lights etc. 

```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in mat4 model_matrix;
layout (location=5) in mat4 inverse_model_matrix;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec3 colourdata_for_the_fragmentshader;

void main() {
    vec4 worldpos = model_matrix*vec4(position,1.0);
    gl_Position = ubo.projection_matrix*ubo.view_matrix*worldpos;
    colourdata_for_the_fragmentshader=vec3(1.0,1.0,0.5);
}
```

```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec3 colour_in;


void main(){
	theColour=vec4(colour_in,1.0);
}
```

Of course, this shader will not remain at the stage of "use just this one fixed colour" — but for now I'm still setting the stage; — and recalling which places I have to change; it has been some time since I've touched this project.

Another crucial place for these changes is the function `Pipeline::init()`, which I copy and modify into a `Pipeline::init_textured()` function as
follows: 

```rust 
    pub fn init_textured(
        logical_device: &ash::Device,
        swapchain: &SwapchainDongXi,
        renderpass: &vk::RenderPass,
    ) -> Result<Pipeline, vk::Result> {
        let vertexshader_createinfo = vk::ShaderModuleCreateInfo::builder().code(
            vk_shader_macros::include_glsl!("./shaders/shader_textured.vert", kind: vert),
        );
        let vertexshader_module =
            unsafe { logical_device.create_shader_module(&vertexshader_createinfo, None)? };
        let fragmentshader_createinfo = vk::ShaderModuleCreateInfo::builder().code(
            vk_shader_macros::include_glsl!("./shaders/shader_textured.frag"),
        );
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
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 6,
                offset: 80,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 7,
                offset: 96,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 8,
                offset: 112,
                format: vk::Format::R32G32B32A32_SFLOAT,
            }
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 12,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 128,
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
            .cull_mode(vk::CullModeFlags::BACK)
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
        let descriptorset_layout_binding_descs0 = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
            .descriptor_count(1)
            .stage_flags(vk::ShaderStageFlags::VERTEX)
            .build()];
        let descriptorset_layout_info0 = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs0);
        let descriptorsetlayout0 = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info0, None)
        }?;

        let desclayouts = vec![descriptorsetlayout0];
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
```
The changes directly correspond to the vertex attributes as above (in the `Vertex...Description`s) or follow from the removal of the lights (with some
lines about `descriptorset` being removed compared to the previous version). 

Naturally, we also call this function instead of the old one in `Aetna::init()`.

In `update_commandbuffer` we remove the line about the lights: 
```rust
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
                &[
                    self.descriptor_sets_camera[index],
//                    self.descriptor_sets_light[index],
                ],
                &[],
            );
            for m in &self.models {
                m.draw(&self.device, commandbuffer);
            }
            self.device.cmd_end_render_pass(commandbuffer);
            self.device.end_command_buffer(commandbuffer)?;
        }
```

Does this run now? 

Not quite: We still refer to the ligths somewhere: In `Aetna::init()`, we should use `descriptor_sets_light: vec![],` in the final struct
initialization, and remove (or, maybe, better only put into a comment) the part relying on the light and its descriptor set layout:
```rust
        /*        let desc_layouts_light =
            vec![pipeline.descriptor_set_layouts[1]; swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info_light = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts_light);
        let descriptor_sets_light = unsafe {
            logical_device.allocate_descriptor_sets(&descriptor_set_allocate_info_light)
        }?;

        for descset in &descriptor_sets_light {
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: lightbuffer.buffer,
                offset: 0,
                range: 8,
            }];
            let desc_sets_write = [vk::WriteDescriptorSet::builder()
                .dst_set(*descset)
                .dst_binding(0)
                .descriptor_type(vk::DescriptorType::STORAGE_BUFFER)
                .buffer_info(&buffer_infos)
                .build()];
            unsafe { logical_device.update_descriptor_sets(&desc_sets_write, &[]) };
        }*/
```

Now it runs. A pale yellow rectangle. We can start to work.

Our goal is to replace the yellowness of the surface and replace it by some image. By a "Texture". First we need an image. Let's create a .png file,
say `gfx/image.png`. The content is not important, it just shouldn't be pale yellow, so that we can see when we are successful in the end. 

Next step: Get the image into our program. Let us write `mod texture;` in `main.rs` and create a new file `texture.rs`: 
```rust
struct Texture {
    image: image::RgbaImage,
}

impl Texture {
    fn from_file<P: AsRef<std::path::Path>>(path: P) -> Self {
        Texture {
            image: image::open(path)
                .map(|img| img.to_rgba())
                .expect("unable to open image"),
        }
    }
}
```
Next step: Get this image to the GPU. We will need a Vulkan image for that, together with corresponding memory (reserved on the GPU and bound to the
image); like in the chapter about screenshots, we rely on the Vulkan Memory Allocator for that.

```rust
struct Texture {
    image: image::RgbaImage,
    vk_image: vk::Image,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
}

impl Texture {
    fn from_file<P: AsRef<std::path::Path>>(path: P, allocator: &vk_mem::Allocator) -> Self {
        let image = image::open(path)
            .map(|img| img.to_rgba())
            .expect("unable to open image");
        let img_create_info = vk::ImageCreateInfo::builder();
        let alloc_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::GpuOnly,
            ..Default::default()
        };
        let (vk_image, allocation, allocation_info) = allocator
            .create_image(&img_create_info, &alloc_create_info)
            .expect("creating vkImage for texture");
        Texture {
            image,
            vk_image,
            allocation,
            allocation_info,
        }
    }
}
```

This `img_create_info` was more of a placeholder. Let us fill in some data, beginning with the size of the image:
```rust
        let (width, height) = image.dimensions();
        let img_create_info = vk::ImageCreateInfo::builder()
            .image_type(vk::ImageType::TYPE_2D)
            .extent(vk::Extent3D {
                width,
                height,
                depth: 1,
            })
            .mip_levels(1)
            .array_layers(1)
            .format(vk::Format::R8G8B8A8_SRGB)
            .samples(vk::SampleCountFlags::TYPE_1)
            .usage(vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED);
```
The important parts are the size (width, height, and a not-really-existent third dimension, hence depth 1), and the usage flags: We want to transfer
data to this image and want to use it for sampling in the shader (see below). 

If we want to use the image as more than a mere array of data, if we want to access it, we additionally need a `vk::ImageView` (just as in our first encounter with `VkImage`s in the swapchain chapter).
```rust 
        let view_create_info = vk::ImageViewCreateInfo::builder()
            .image(vk_image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::R8G8B8A8_SRGB)
            .subresource_range(vk::ImageSubresourceRange {
                aspect_mask: vk::ImageAspectFlags::COLOR,
                level_count: 1,
                layer_count: 1,
                ..Default::default()
            });
        let imageview = unsafe { device.create_image_view(&view_create_info, None) }
            .expect("image view creation");
```
(It's a view for the image we have just created, it's 2D with the same format as always and we're interested in the colour and there's only one layer.) The creation needs access to the logical device (which we'll pass as another argument to the function), and we add `imageview` as another field to `Texture`.

This is - still! - not enough. We will be interested in reading the colour from this image(view). Like: "Hey, GPU! What's the colour exactly in the
middle of this imageview? Please draw the same colour at the following position." What we are passing in is, essentially, a grid of colour values. Now if "exactly in the middle" is not on this grid (is not a pixel (if we're thinking of pixels as points and not as small squares) of our image), how should our poor GPU respond? Should it use the closest value? Should it take all neighbouring values and interpolate? 

Well, somehow it's our job to decide that. And the structure to record this decision is a Sampler. (I see a new field for the Texture struct ...)
```rust
        let sampler_info = vk::SamplerCreateInfo::builder()
            .mag_filter(vk::Filter::LINEAR)
            .min_filter(vk::Filter::LINEAR);
        let sampler =
            unsafe { device.create_sampler(&sampler_info, None) }.expect("sampler creation");
```
Here we're saying that values should be linearly interpolated. (Another option would have been `NEAREST`.) 

Now we have created the structures, but we have not sent the content of the image to the GPU yet.

Let us prepare a buffer with this data: 
```rust
        let data = image.clone().into_raw();
        let mut buffer = Buffer::new(
            &aetna.allocator,
            data.len() as u64,
            vk::BufferUsageFlags::TRANSFER_SRC,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer.fill(&aetna.allocator, &data);
```
Next, there should be a commandbuffer so that we can actually send some commands.
```rust
        let commandbuf_allocate_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(aetna.pools.commandpool_graphics)
            .command_buffer_count(1);
        let copycmdbuffer = unsafe {
            aetna
                .device
                .allocate_command_buffers(&commandbuf_allocate_info)
        }
        .unwrap()[0];

        let cmdbegininfo = vk::CommandBufferBeginInfo::builder()
            .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);
        unsafe {
            aetna
                .device
                .begin_command_buffer(copycmdbuffer, &cmdbegininfo)
        }?;
	
	//Insert commands here.
 
        unsafe { aetna.device.end_command_buffer(copycmdbuffer) }?;
        let submit_infos = [vk::SubmitInfo::builder()
            .command_buffers(&[copycmdbuffer])
            .build()];
        let fence = unsafe {
            aetna
                .device
                .create_fence(&vk::FenceCreateInfo::default(), None)
        }?;
        unsafe {
            aetna
                .device
                .queue_submit(aetna.queues.graphics_queue, &submit_infos, fence)
        }?;
        unsafe { aetna.device.wait_for_fences(&[fence], true, std::u64::MAX) }?;
        unsafe { aetna.device.destroy_fence(fence, None) };
        aetna.allocator.destroy_buffer(buffer.buffer, &buffer.allocation)?;
        unsafe {
            aetna
                .device
                .free_command_buffers(aetna.pools.commandpool_graphics, &[copycmdbuffer])
        };
```
As to `//Insert commands here`: What we want is copying data from the buffer into the image. The command to use is `cmd_copy_buffer_to_image`; that
sounds reasonable. We also have to describe which regions (of buffer and image) we want to affect:

```rust
       let image_subresource = vk::ImageSubresourceLayers {
            aspect_mask: vk::ImageAspectFlags::COLOR,
            mip_level: 0,
            base_array_layer: 0,
            layer_count: 1,
        };
        let region = vk::BufferImageCopy {
            buffer_offset: 0,
            buffer_row_length: 0,
            buffer_image_height: 0,
            image_offset: vk::Offset3D { x: 0, y: 0, z: 0 },
            image_extent: vk::Extent3D {
                width,
                height,
                depth: 1,
            },
            image_subresource,
            ..Default::default()
        };
        unsafe {
            aetna.device.cmd_copy_buffer_to_image(
                copycmdbuffer,
                buffer.buffer,
                vk_image,
                vk::ImageLayout::TRANSFER_DST_OPTIMAL,
                &[region],
            );
        }
```
This gives 
```
Submitted command buffer expects VkImage 0x290000000029[] (subresource: aspectMask 0x1 array layer 0, mip level 0) to be in layout VK_IMAGE_LAYOUT_TRAN
SFER_DST_OPTIMAL--instead, current layout is VK_IMAGE_LAYOUT_UNDEFINED.
```
Right. The copy command we submitted includes `vk::ImageLayout::TRANSFER_DST_OPTIMAL`. (This is not the only possible value, but `LAYOUT_UNDEFINED` is
no admissible choice here.) Therefore, we have to prepare the image; we set a MemoryBarrier for that purpose (just at the beginning of the command
buffer): 
```rust
        let barrier = vk::ImageMemoryBarrier::builder()
            .image(vk_image)
            .src_access_mask(vk::AccessFlags::empty())
            .dst_access_mask(vk::AccessFlags::TRANSFER_WRITE)
            .old_layout(vk::ImageLayout::UNDEFINED)
            .new_layout(vk::ImageLayout::TRANSFER_DST_OPTIMAL)
            .subresource_range(vk::ImageSubresourceRange {
                aspect_mask: vk::ImageAspectFlags::COLOR,
                base_mip_level: 0,
                level_count: 1,
                base_array_layer: 0,
                layer_count: 1,
            })
            .build();
        unsafe {
            aetna.device.cmd_pipeline_barrier(
                copycmdbuffer,
                vk::PipelineStageFlags::TOP_OF_PIPE,
                vk::PipelineStageFlags::TRANSFER,
                vk::DependencyFlags::empty(),
                &[],
                &[],
                &[barrier],
            )
        };
```
That gets rid of the error message. But seeing that transferring data into the image is not our final purpose, let us change the image layout once
more (after the copy command) to `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`:
```rust
        let barrier = vk::ImageMemoryBarrier::builder()
            .image(vk_image)
            .src_access_mask(vk::AccessFlags::TRANSFER_WRITE)
            .dst_access_mask(vk::AccessFlags::SHADER_READ)
            .old_layout(vk::ImageLayout::TRANSFER_DST_OPTIMAL)
            .new_layout(vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL)
            .subresource_range(vk::ImageSubresourceRange {
                aspect_mask: vk::ImageAspectFlags::COLOR,
                base_mip_level: 0,
                level_count: 1,
                base_array_layer: 0,
                layer_count: 1,
            })
            .build();
        unsafe {
            aetna.device.cmd_pipeline_barrier(
                copycmdbuffer,
                vk::PipelineStageFlags::TRANSFER,
                vk::PipelineStageFlags::FRAGMENT_SHADER,
                vk::DependencyFlags::empty(),
                &[],
                &[],
                &[barrier],
            )
        };
```
Now the image should be on the GPU. But we're not using it yet. 

It is some amount of data that is shared by all vertices (well, for just showing the image, "all" may be "four" — but it certainly doesn't change from
one vertex to the next), so it has some similarity with the camera data or the light buffer we had earlier. We should, hence, prepare a descriptor set
binding during pipeline setup (function `init_textured`). 
```rust
        let descriptorset_layout_binding_descs1 = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(1)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
        let descriptorset_layout_info1 = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs1);
        let descriptorsetlayout1 = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info1, None)
        }?;
        let desclayouts = vec![descriptorsetlayout0,descriptorsetlayout1];
```
(The last line is modified, everything else is new.) We will use the image in the fragment shader, and the type is a combined image and sampler. 

When setting up `aetna`, we had a place to specify the size of descriptor pool. Here we now need place for some descriptors of type
`COMBINED_IMAGE_SAMPLER` and thus insert 
```rust
            vk::DescriptorPoolSize {
                ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
                descriptor_count: swapchain.amount_of_images,
            },
```
We also grant `Aetna` a new field `descriptor_sets_texture`, filling it as follows: 
```rust
        let desc_layouts_texture =
            vec![pipeline.descriptor_set_layouts[1]; swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info_texture = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts_texture);
        let descriptor_sets_texture = unsafe {
            logical_device.allocate_descriptor_sets(&descriptor_set_allocate_info_texture)
        }?;
```
We also have to bind these descriptor sets when we update the commandbuffer (in `aetna.update_commandbuffer`):
```rust
            self.device.cmd_bind_descriptor_sets(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline.layout,
                0,
                &[
                    self.descriptor_sets_camera[index],
                    self.descriptor_sets_texture[index],
                ],
                &[],
            );
```
What's left to do is to bring the descriptor set and the image and sampler (from loading the texture) together.

Let's say, we create the texture (close to the beginning of our main function) by
```rust
    let texture = Texture::from_file("../gfx/image.png", &aetna)?;
```
Then we insert 
```rust
            let imageinfo = vk::DescriptorImageInfo {
                image_layout: vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
                image_view: texture.imageview,
                sampler: texture.sampler,
                ..Default::default()
            };
            let descriptorwrite_image = vk::WriteDescriptorSet {
                dst_set: aetna.descriptor_sets_texture[aetna.swapchain.current_image],
                dst_binding: 0,
                dst_array_element: 0,
                descriptor_type: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
                descriptor_count: 1,
                p_image_info: [imageinfo].as_ptr(),
                ..Default::default()
            };

            unsafe {
                aetna
                    .device
                    .update_descriptor_sets(&[descriptorwrite_image], &[]);
            }
```
just before we update the command buffer. 

Now the image should find its way to the GPU and be accessible during the fragment shader stage.

Time to enter the fragment shader and introduce the sampler: 
```glsl
layout(set=1,binding=0) uniform sampler2D texturesampler;
```
How do we access it? This way: 
```glsl
	theColour=texture(texturesampler,vec2(0.5,0.5));
```
The `vec2` gives the coordinates; here we are using the midpoint of the image. (And if we run the program, the rect is no longer yellow.) 

Of course, with this line the whole rect has the same colour. If we want it to resemble the image, we will have to vary the texture coordinates and
use the whole range from 0 to 1 instead of only 0.5.

The best way to do that is to give each vertex its texture coordinates as vertex data, have them (interpolated and) passed to the fragment shader, and
use them to read from the correct place of the texture. 

With x, y, z being reserved for the spatial coordinates, it is usual to use u and v as names for the texture coordinates. 

That makes our fragment shader the following: 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec2 uv;

layout(set=1,binding=0) uniform sampler2D texturesampler;

void main(){
	theColour=texture(texturesampler,uv);
}
```
(I have removed the colour data that was passed here previously. Colour is supposed to originate from the texture only.) 

```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in vec2 texcoord;
layout (location=2) in mat4 model_matrix;
layout (location=6) in mat4 inverse_model_matrix;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec2 uv;

void main() {
    vec4 worldpos = model_matrix*vec4(position,1.0);
    gl_Position = ubo.projection_matrix*ubo.view_matrix*worldpos;
    uv = texcoord;
}
```
This is the vertex shader, now getting the texture coordinate as one property of the vertices. Note the changes in `location` values. 

That means a small change to the VertexAttributeDescriptions:
```rust
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                offset: 12,
                format: vk::Format::R32G32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 2,
                offset: 0,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 3,
                offset: 16,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 4,
                offset: 32,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 5,
                offset: 48,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 6,
                offset: 64,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 7,
                offset: 80,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 8,
                offset: 96,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 9,
                offset: 112,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 20,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 128,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
```
And we have to add the texture coordinates as part of the VertexData struct: 

```rust
#[derive(Copy, Clone, Debug)]
#[repr(C)]
pub struct TexturedVertexData {
    pub position: [f32; 3],
    pub texcoord: [f32;2],
}
```
Also, the `quad` creation assigns these values: 
```rust
    pub fn quad() -> Self {
        let lb = TexturedVertexData {
            position: [-1.0, 1.0, 0.0],
            texcoord: [0.0, 1.0],
        }; //lb: left-bottom
        let lt = TexturedVertexData {
            position: [-1.0, -1.0, 0.0],
            texcoord: [0.0, 0.0],
        };
        let rb = TexturedVertexData {
            position: [1.0, 1.0, 0.0],
            texcoord: [1.0, 1.0],
        };
        let rt = TexturedVertexData {
            position: [1.0, -1.0, 0.0],
            texcoord: [1.0, 0.0],
        };
```

And with that, our picture can be seen on the screen. 


[Continue](038_Textures.md)
