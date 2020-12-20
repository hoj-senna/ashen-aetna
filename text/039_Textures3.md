## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Text(ures) 

Now that we have the means to bring a picture to the screen, let us take care of drawing text. Turning strings into correct shapes is a nontrivial
task and we will assume that some other library has already done this. 

Let us assume that for every letter we know the position where to draw it, the size (width and height in pixels) and have an array telling us which
pixels belong to the letter. (Like "3x3, [1,1,1,1,0,1,1,1,1]" as a very crude approximation of an "o". Except that we will not use 0-1, but 0-255 so
as to cope with pixels partially being part of the letter.) We can also give the text (each letter) a colour. 

That makes a small difference in the textures we use (different, smaller colour format), and, of course, in the shaders. However, the drawing itself
should be close to that of the last two chapters. (Let me note that there are other ways of dealing with text, especially if some of the preparatory
steps are also transferred to the GPU, but for now I'd like to stick with the easiest one. — I might revisit the topic in a later chapter, if I ever
get to it.)


What will we need in the shaders? 
- As uniforms, the textures carrying the letters' shape information. We do not need the camera info if we only care about drawing to the screen. 
- As vertex attributes, position (2d would suffice, but let's use 3d — that could make having texts behind each other easier), texture coordinates,
  and colour. Also, texture id. 

In contrast to the general (and possibly complex) models we targeted earlier, all "instances" would only consist of four vertices and instanced or
indexed rendering will not come with a large performance win (possibly even a performance loss, due to overhead). Not that this plays a major role,
but I feel that text is sufficiently special to not be part of our previous `Model` construct.

Let us first invent a suitable texture struct (as the first lines to enter a new `text.rs`): 

```rust
pub struct TextTexture {
    vk_image: vk::Image,
    pub imageview: vk::ImageView,
    pub sampler: vk::Sampler,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
}
```

This time, I'm not including an `RgbaImage`. Creating these textures (from a slice of `u8`s and their width and height in pixels) then works as
follows (and the differences to previous texture creation should be small: `data` are passed in and not taken from the image): 


```rust
impl TextTexture {
    pub fn from_u8s(
        data: &[u8],
        width: u32,
        height: u32,
        device: &ash::Device,
        allocator: &vk_mem::Allocator,
        commandpool_graphics: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let img_create_info = vk::ImageCreateInfo::builder()
            .image_type(vk::ImageType::TYPE_2D)
            .extent(vk::Extent3D {
                width,
                height,
                depth: 1,
            })
            .mip_levels(1)
            .array_layers(1)
            .format(vk::Format::R8_SRGB)
            .samples(vk::SampleCountFlags::TYPE_1)
            .usage(vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED);
        let alloc_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::GpuOnly,
            ..Default::default()
        };
        let (vk_image, allocation, allocation_info) = allocator
            .create_image(&img_create_info, &alloc_create_info)
            .expect("creating vkImage for texture");
        let view_create_info = vk::ImageViewCreateInfo::builder()
            .image(vk_image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(vk::Format::R8_SRGB)
            .subresource_range(vk::ImageSubresourceRange {
                aspect_mask: vk::ImageAspectFlags::COLOR,
                level_count: 1,
                layer_count: 1,
                ..Default::default()
            });
        let imageview = unsafe { device.create_image_view(&view_create_info, None) }
            .expect("image view creation");
        let sampler_info = vk::SamplerCreateInfo::builder()
            .mag_filter(vk::Filter::LINEAR)
            .min_filter(vk::Filter::LINEAR);
        let sampler =
            unsafe { device.create_sampler(&sampler_info, None) }.expect("sampler creation");

        let mut buffer = Buffer::new(
            allocator,
            data.len() as u64,
            vk::BufferUsageFlags::TRANSFER_SRC,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer.fill(allocator, &data);

        let commandbuf_allocate_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(*commandpool_graphics)
            .command_buffer_count(1);
        let copycmdbuffer =
            unsafe { device.allocate_command_buffers(&commandbuf_allocate_info) }.unwrap()[0];

        let cmdbegininfo = vk::CommandBufferBeginInfo::builder()
            .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);
        unsafe { device.begin_command_buffer(copycmdbuffer, &cmdbegininfo) }?;

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
            device.cmd_pipeline_barrier(
                copycmdbuffer,
                vk::PipelineStageFlags::TOP_OF_PIPE,
                vk::PipelineStageFlags::TRANSFER,
                vk::DependencyFlags::empty(),
                &[],
                &[],
                &[barrier],
            )
        };

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
            device.cmd_copy_buffer_to_image(
                copycmdbuffer,
                buffer.buffer,
                vk_image,
                vk::ImageLayout::TRANSFER_DST_OPTIMAL,
                &[region],
            );
        }

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
            device.cmd_pipeline_barrier(
                copycmdbuffer,
                vk::PipelineStageFlags::TRANSFER,
                vk::PipelineStageFlags::FRAGMENT_SHADER,
                vk::DependencyFlags::empty(),
                &[],
                &[],
                &[barrier],
            )
        };

        unsafe { device.end_command_buffer(copycmdbuffer) }?;
        let submit_infos = [vk::SubmitInfo::builder()
            .command_buffers(&[copycmdbuffer])
            .build()];
        let fence = unsafe { device.create_fence(&vk::FenceCreateInfo::default(), None) }?;
        unsafe { device.queue_submit(*graphics_queue, &submit_infos, fence) }?;
        unsafe { device.wait_for_fences(&[fence], true, std::u64::MAX) }?;
        unsafe { device.destroy_fence(fence, None) };
        allocator.destroy_buffer(buffer.buffer, &buffer.allocation)?;
        unsafe { device.free_command_buffers(*commandpool_graphics, &[copycmdbuffer]) };

        Ok(Texture {
            image,
            vk_image,
            imageview,
            sampler,
            allocation,
            allocation_info,
        })
    }
```

For drawing let's use the following shaders: 

As vertex shader `shaders/text.vert`: 

```glsl
#version 450

layout (location=0) in vec3 in_position;
layout (location=1) in vec2 in_texcoord;
layout (location=2) in vec3 in_colour;
layout (location=3) in uint in_texture_id;

layout (location=0) out vec2 out_texcoord;
layout (location=1) out vec3 out_colour;
layout (location=2) out uint out_texture_id;

void main() {
	gl_Position=vec4(in_position,1.0);
	out_texcoord=in_texcoord;
	out_colour=in_colour;
	out_texture_id=in_texture_id;
}
```
and as `shaders/text.frag`: 
```glsl
#version 450
#extension GL_EXT_nonuniform_qualifier : require

layout (location=0) out vec4 theColour;

layout (location=0) in vec2 texcoord;
layout (location=1) in vec3 colour;
layout (location=2) flat in uint texture_id;


layout(set=0,binding=0) uniform sampler2D lettertextures[];

void main(){
	theColour=vec4(colour,texture(lettertextures[texture_id],texcoord));
}
```
These are fairly minimal and by now probably require no explanation. 

Of course, we need some pipeline which consists of these shaders: 



`Pipeline::init_text()`: 
```rust
    pub fn init_text(
        logical_device: &ash::Device,
        swapchain: &SwapchainDongXi,
        renderpass: &vk::RenderPass,
    ) -> Result<Pipeline, vk::Result> {
        let vertexshader_createinfo = vk::ShaderModuleCreateInfo::builder().code(
            vk_shader_macros::include_glsl!("./shaders/text.vert", kind: vert),
        );
        let vertexshader_module =
            unsafe { logical_device.create_shader_module(&vertexshader_createinfo, None)? };
        let fragmentshader_createinfo = vk::ShaderModuleCreateInfo::builder()
            .code(vk_shader_macros::include_glsl!("./shaders/text.frag"));
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
                binding: 0,
                location: 1,
                offset: 12,
                format: vk::Format::R32G32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 2,
                offset: 20,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 3,
                offset: 32,
                format: vk::Format::R8G8B8A8_UINT,
            },
        ];
        let vertex_binding_descs = [vk::VertexInputBindingDescription {
            binding: 0,
            stride: 36,
            input_rate: vk::VertexInputRate::VERTEX,
        }];
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
        let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(MAXIMAL_NUMBER_OF_TEXTURES)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
        let descriptor_binding_flags = [vk::DescriptorBindingFlags::VARIABLE_DESCRIPTOR_COUNT];
        let mut descriptorset_layout_binding_flags =
            vk::DescriptorSetLayoutBindingFlagsCreateInfo::builder()
                .binding_flags(&descriptor_binding_flags);
        let descriptorset_layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs)
            .push_next(&mut descriptorset_layout_binding_flags);
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
```

Now where should we get our letters from? We could hardcode the shapes of every ASCII character, for example, or just use one large texture that
already has every letter on it. That's feasible for alphabet languages and as long as we only want to use a small number of different fonts (and maybe
sizes). Or we could use a different crate that loads a font file and upon being given a string and a text size tells us where to place the letters and
how they look. I want to employ `fontdue`; let's enlarge `Cargo.toml`:

```rust
fontdue = "0.2.4"
```
and we can begin creating a struct that takes care of everything text-related:
```rust
pub struct AllText {
    fonts: Vec<fontdue::Font>,
}
```
There will be more fields, don't worry. But first let's make it possible to load a font: 

```rust
    pub fn load_font<P: AsRef<std::path::Path>>(
        &mut self,
        path: P,
    ) -> Result<usize, Box<dyn std::error::Error>> {
        let fontdata = std::fs::read(path)?;
        let font = fontdue::Font::from_bytes(fontdata, fontdue::FontSettings::default())?;
        self.fonts.push(font);
        Ok(self.fonts.len() - 1)
    }
```
We have a font, we can prepare letters. Maybe in this way: 

```rust
pub struct Letter {
    colour: [f32; 3],
    position_and_shape: fontdue::layout::GlyphPosition,
}
```
As you see, I want to give them a colour; in `position_and_shape` we store all other important information (as already intended by fontdue), in
particular position and shape, of course. 

We will create letters in the following way. (Here, `text` is an `AllText` struct that becomes a member of our `Aetna`.) The `TextStyle` contains a
String, text size, and the index of the font to use (and the latter is the reason why `create_letters` becomes a method of `AllText`); the second
argument of `create_letters` is the colour: 

```rust
    let letters = aetna.text.create_letters(
        &[
            &fontdue::layout::TextStyle::new("Hello ", 35.0, 0),
            &fontdue::layout::TextStyle::new("world!", 40.0, 0),
        ],
        [0., 1., 0.],
    );
```
The function looks as follows. The best description is probably: Let fontdue take care of the work.
```rust
    pub fn create_letters(
        &self,
        styles: &[&fontdue::layout::TextStyle],
        colour: [f32; 3],
    ) -> Vec<Letter> {
        let mut layout = fontdue::layout::Layout::new();
        let mut output = Vec::new();
        let settings = fontdue::layout::LayoutSettings {
            ..fontdue::layout::LayoutSettings::default()
        };
        layout.layout_horizontal(&self.fonts, styles, &settings, &mut output);
        let mut letters: Vec<Letter> = vec![];
        for glyph in output {
            letters.push(Letter {
                colour,
                position_and_shape: glyph,
            });
        }
        letters
    }
```

Okay, with that we have letters. We have to turn them into something that we can display. In the end, that means, we must produce textures (we have
prepared the function for that) and a vertex buffer. 

Let us enhance the `AllText` struct. In addition to the fonts, we put in the textures. If a letter appears twice (in the same size and font), we'll,
however, not create a second texture, but reuse one. For this purpose, we include a `HashMap` translating `GlyphRasterConfig` (i.e. character, scale, font) to the corresponding texture. 
We also include vertex data (to be created from letters) and a vertexbuffer, as well as the pipeline and descriptors: 

```rust
pub struct AllText {
    vertexdata: Vec<TextVertexData>,
    vertexbuffer: Option<Buffer>,
    textures: Vec<TextTexture>,
    texture_ids: HashMap<fontdue::layout::GlyphRasterConfig, u32>,
    fonts: Vec<fontdue::Font>,
    pipeline: Option<Pipeline>,
    desc_pool: Option<vk::DescriptorPool>,
    descriptorsets: Vec<vk::DescriptorSet>,
}
```
As to the vertex format: 
```rust
#[derive(Copy, Clone, Debug)]
#[repr(C)]
pub struct TextVertexData {
    pub position: [f32; 3],
    pub texcoord: [f32; 2],
    pub colour: [f32; 3],
    pub texture_id: u32,
}
```
Translating letters to vertex positions happens in this function: 
```rust
    pub fn create_vertexdata(
        &mut self,
        letters: Vec<Letter>,
        position: (u32, u32), //in pixels
        window: &winit::window::Window,
        device: &ash::Device,
        allocator: &vk_mem::Allocator,
        commandpool_graphics: &vk::CommandPool,
        graphics_queue: &vk::Queue,
        swapchain: &SwapchainDongXi,
    ) {
        let screensize = window.inner_size();
        let mut need_texture_update = false;
        let mut vertexdata = Vec::with_capacity(6 * letters.len());
        for l in letters {
            let id_option = self.texture_ids.get(&l.position_and_shape.key);
            let id;
            if id_option.is_none() {
                let (metrics, bitmap) = self.fonts[l.position_and_shape.key.font_index]
                    .rasterize(l.position_and_shape.key.c, l.position_and_shape.key.px);
                need_texture_update = true;
                id = self
                    .new_texture_from_u8s(
                        &bitmap,
                        metrics.width as u32,
                        metrics.height as u32,
                        device,
                        allocator,
                        commandpool_graphics,
                        graphics_queue,
                    )
                    .unwrap() as u32;
                self.texture_ids.insert(l.position_and_shape.key, id);
            } else {
                id = *id_option.unwrap() as u32;
            }
            let left =
                2. * (l.position_and_shape.x + position.0 as f32) / screensize.width as f32 - 1.0;
            let right = 2.
                * (l.position_and_shape.x + position.0 as f32 + l.position_and_shape.width as f32)
                / screensize.width as f32
                - 1.0;
            let top = 2.
                * (-l.position_and_shape.y + position.1 as f32
                    - l.position_and_shape.height as f32)
                / screensize.height as f32
                - 1.0;
            let bottom =
                2. * (-l.position_and_shape.y + position.1 as f32) / screensize.height as f32 - 1.0;
            let v1 = TextVertexData {
                position: [left, top, 0.],
                texcoord: [0., 0.],
                colour: l.colour,
                texture_id: id,
            };
            let v2 = TextVertexData {
                position: [left, bottom, 0.],
                texcoord: [0., 1.],
                colour: l.colour,
                texture_id: id,
            };
            let v3 = TextVertexData {
                position: [right, top, 0.],
                texcoord: [1., 0.],
                colour: l.colour,
                texture_id: id,
            };
            let v4 = TextVertexData {
                position: [right, bottom, 0.],
                texcoord: [1., 1.],
                colour: l.colour,
                texture_id: id,
            };
            vertexdata.push(v1);
            vertexdata.push(v2);
            vertexdata.push(v3);
            vertexdata.push(v3);
            vertexdata.push(v2);
            vertexdata.push(v4);
        }
        self.vertexdata.append(&mut vertexdata);
        if need_texture_update {
            self.update_textures(swapchain, device);
        }
    }
```
If the letter already has some texture, we only compute the vertex positions from given coordinates and the info obtained from fontdue. If the letter
is new, we create a new texture
```rust
    pub fn new_texture_from_u8s(
        &mut self,
        data: &[u8],
        width: u32,
        height: u32,
        device: &ash::Device,
        allocator: &vk_mem::Allocator,
        commandpool_graphics: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> Result<usize, Box<dyn std::error::Error>> {
        let new_texture = TextTexture::from_u8s(
            data,
            width,
            height,
            device,
            allocator,
            commandpool_graphics,
            graphics_queue,
        )?;
        let new_id = self.textures.len();
        self.textures.push(new_texture);
        Ok(new_id)
    }
```
(which boils down to passing all arguments through to the first function in this chapter) and at the end update the textures: 
```rust
    pub fn update_textures(
        &mut self,
        swapchain: &SwapchainDongXi,
        device: &ash::Device,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let amount = self.textures.len() as u32;
        if let Some(pool) = self.desc_pool {
            unsafe {
                device.destroy_descriptor_pool(pool, None);
            }
        }
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
           descriptor_count: crate::renderpass_and_pipeline::MAXIMAL_NUMBER_OF_TEXTURES
                * swapchain.amount_of_images,
        }];
        let descriptor_pool_info = vk::DescriptorPoolCreateInfo::builder()
            .max_sets(swapchain.amount_of_images) 
            .pool_sizes(&pool_sizes);
        let descriptor_pool =
            unsafe { device.create_descriptor_pool(&descriptor_pool_info, None) }?;
        self.desc_pool = Some(descriptor_pool);

        let desc_layouts_text = vec![
            self.pipeline.as_ref().unwrap().descriptor_set_layouts[0];
            swapchain.amount_of_images as usize
        ];
        let descriptor_set_allocate_info_text = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(descriptor_pool)
            .set_layouts(&desc_layouts_text);
        let descriptor_sets_text =
            unsafe { device.allocate_descriptor_sets(&descriptor_set_allocate_info_text) }?;

        for i in 0..swapchain.amount_of_images {
            let imageinfos: Vec<vk::DescriptorImageInfo> = self
                .textures
                .iter()
                .map(|t| vk::DescriptorImageInfo {
                    image_layout: vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
                    image_view: t.imageview,
                    sampler: t.sampler,
                    ..Default::default()
                })
                .collect();

            let descriptorwrite_image = vk::WriteDescriptorSet::builder()
                .dst_set(descriptor_sets_text[i as usize])
                .dst_binding(0)
                .dst_array_element(0)
                .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
                .image_info(&imageinfos)
                .build();
            unsafe {
                device.update_descriptor_sets(&[descriptorwrite_image], &[]);
            }
        }
        self.descriptorsets = descriptor_sets_text;
        Ok(())
    }
```
The function `create_vertexdata` only created some `TextVertexData`, we still have to turn that into a usable buffer: 
```rust
    pub fn update_vertexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if self.vertexdata.is_empty() {
            return Ok(());
        }
        if let Some(buffer) = &mut self.vertexbuffer {
            buffer.fill(allocator, &self.vertexdata)?;
            Ok(())
        } else {
            let bytes = (self.vertexdata.len() * std::mem::size_of::<TextVertexData>()) as u64;
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
```
And, by the way, we should have a way to create an `AllText`. At that time, we already decide on a stardard font:
```rust
    pub fn new<P: AsRef<std::path::Path>>(
        device: &ash::Device,
        swapchain: &SwapchainDongXi,
        renderpass: &vk::RenderPass,
        standardfont: P,
    ) -> Result<AllText, Box<dyn std::error::Error>> {
        let pip = Pipeline::init_text(device, swapchain, renderpass)?;

        let mut all_text = AllText {
            vertexdata: vec![],
            vertexbuffer: None,
            textures: vec![],
            texture_ids: HashMap::new(),
            fonts: vec![],
            pipeline: Some(pip),
            desc_pool: None,
            descriptorsets: vec![],
        };
        all_text.load_font(standardfont);
        Ok(all_text)
    }
```
and cleanup:
```rust
    pub fn cleanup(&mut self, device: &ash::Device, allocator: &vk_mem::Allocator) {
        let p = self.pipeline.take();
        if let Some(pip) = p {
            pip.cleanup(device);
        }
        let p = self.desc_pool.take();
        if let Some(pool) = p {
            unsafe {
                device.destroy_descriptor_pool(pool, None);
            }
        }
        let b = self.vertexbuffer.take();
        if let Some(buf) = b {
            unsafe { allocator.destroy_buffer(buf.buffer, &buf.allocation) };
        }
        for texture in &self.textures {
            unsafe {
                device.destroy_sampler(texture.sampler, None);
                device.destroy_image_view(texture.imageview, None);
            }
            allocator.destroy_image(texture.vk_image, &texture.allocation);
        }
    }
```
One part is — obviously — still missing: We should also draw the text at some time. Let's prepare the method for that: 
Setting pipeline, descriptor set and vertexbuffer and then drawing. (By the way, `index` refers to the image index that we get during the
`RedrawRequested` branch of the main event loop.) 
```rust
    pub fn draw(
        &self,
        logical_device: &ash::Device,
        commandbuffer: vk::CommandBuffer,
        index: usize,
    ) {
        if let Some(pipeline) = &self.pipeline {
            if self.descriptorsets.len() > index {
                unsafe {
                    logical_device.cmd_bind_pipeline(
                        commandbuffer,
                        vk::PipelineBindPoint::GRAPHICS,
                        pipeline.pipeline,
                    );

                    logical_device.cmd_bind_descriptor_sets(
                        commandbuffer,
                        vk::PipelineBindPoint::GRAPHICS,
                        pipeline.layout,
                        0,
                        &[self.descriptorsets[index]],
                        &[],
                    );
                }

                if let Some(vertexbuffer) = &self.vertexbuffer {
                    unsafe {
                        logical_device.cmd_bind_vertex_buffers(
                            commandbuffer,
                            0,
                            &[vertexbuffer.buffer],
                            &[0],
                        );
                        logical_device.cmd_draw(
                            commandbuffer,
                            self.vertexdata.len() as u32,
                            1, //instance count
                            0,
                            0,
                        );
                    }
                }
            }
        }
    }
```

In `aetna::update_commandbuffer` we include one line (the one about `self.text.draw`):
```rust
            for m in &self.models {
                m.draw(&self.device, commandbuffer);
            }
            self.text.draw(&self.device, commandbuffer, index);
            self.device.cmd_end_render_pass(commandbuffer);
            self.device.end_command_buffer(commandbuffer)?;
        }
        Ok(())
    }
```
And `aetna` gets a new field `text` (as mentioned before), which we initialize by 
```rust
        let text = AllText::new(
            &logical_device,
            &swapchain,
            &renderpass,
            "../fonts/Roboto-Regular.ttf",
        )?;
```
(or with any other font of your choice). Also `Aetna::drop` receives a new line: 

```rust
 self.text.cleanup(&self.device, &self.allocator);
```

Somewhere near the beginning of `main()` (after creating `aetna`, of course):
```rust
    let letters = aetna.text.create_letters(
        &[
            &fontdue::layout::TextStyle::new("Hello ", 35.0, 0),
            &fontdue::layout::TextStyle::new("world!", 40.0, 0),
            &fontdue::layout::TextStyle::new("(and smaller)", 8.0, 0),
        ],
        [0., 1., 0.],
    );
    aetna.text.create_vertexdata(
        letters,
        (100, 200),
        &aetna.window,
        &aetna.device,
        &aetna.allocator,
        &aetna.pools.commandpool_graphics,
        &aetna.queues.graphics_queue,
        &aetna.swapchain,
    );
    aetna.text.update_vertexbuffer(&aetna.allocator);
    let letters2 = aetna.text.create_letters(
        &[
            &fontdue::layout::TextStyle::new("Hello ", 35.0, 0),
            &fontdue::layout::TextStyle::new("world!", 40.0, 0),
            &fontdue::layout::TextStyle::new("(and smaller)", 8.0, 0),
        ],
        [0.6, 0.6, 0.6],
    );
    aetna.text.create_vertexdata(
        letters2,
        (100, 400),
        &aetna.window,
        &aetna.device,
        &aetna.allocator,
        &aetna.pools.commandpool_graphics,
        &aetna.queues.graphics_queue,
        &aetna.swapchain,
    );
    aetna.text.update_vertexbuffer(&aetna.allocator);
```
It works. And it looks okay. The trick there is that I'm using rather large text sizes. With the 8 point string, it's a bit less clear. Probably the
approach "single texture per letter" does not work well enough (or maybe it gets better if we revisit the translation from pixels to Vulkan's
coordinates (adding half a pixel somewhere to counter rounding errors?)).

The Descriptor Pools contain (the maximal number of textures) many descriptor sets. See `update_textures`: 
```rust
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
           descriptor_count: crate::renderpass_and_pipeline::MAXIMAL_NUMBER_OF_TEXTURES
                * swapchain.amount_of_images,
        }];
```
This had to be the case, because the number depends on what we decided during pipeline creation. If we use `amount` in place of
`MAXIMAL_NUMBER_OF_TEXTURES`, the validation layers complain. 

We could instead start with lower numbers and recreate the pipeline whenever we surpass the current pool size. That means recreating the pipeline
(which is an operation that is said to be expensive. But how often will we need a letter that we have never used before? 

We give `AllText` a new field: 

   ```rust
 texture_amount: usize, 
```
which we initialize to `0`. And, speaking of initialization: We can set `pipeline` to `None` there. (Then `new` will need only (`self` and) the standard font as argument.)
`update_textures` is where creating a pipeline will happen now (at least sometimes): 

```rust
    pub fn update_textures(
        &mut self,
        swapchain: &SwapchainDongXi,
        device: &ash::Device,
        renderpass: &vk::RenderPass,
    ) -> Result<(), Box<dyn std::error::Error>> {
       let amount = self.textures.len();
        if amount > self.texture_amount {
            self.texture_amount = amount;
            let p = self.pipeline.take();
            if let Some(pip) = p {
                pip.cleanup(device);
            }
        }
        if self.pipeline.is_none() {
            let pip = Pipeline::init_text(device, swapchain, renderpass, amount as u32)?;
            self.pipeline = Some(pip);
        }
        if let Some(pool) = self.desc_pool {
            unsafe {
                device.destroy_descriptor_pool(pool, None);
            }
        }
        let pool_sizes = [vk::DescriptorPoolSize {
            ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
            descriptor_count:  amount as u32 * swapchain.amount_of_images,
        }];
```
Note the descriptor count in the last line. More notes:  The function needs `renderpass` as additional argument (which requires us to make the field `renderpass` of `aetna` a `pub` field. The
`Pipeline::init_text` function gets the number of textures as argument. Change in the function body: 
```rust
        let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(amount_of_textures) //MAXIMAL_NUMBER_OF_TEXTURES)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
```
Okay. (No idea if that has a big effect on performance. But it feels better.) 

The program spits out a lot of validation errors if we resize the window. 


To take care of that, in `aetna::recreate_swapchain` we insert the following lines: 
```rust
        self.text.clear_pipeline(&self.device, &self.allocator);
        self.text
            .update_textures(&self.swapchain, &self.device, &self.renderpass); 
```
Here I have split `AllText::cleanup()` in two:
```rust
    pub fn clear_pipeline(&mut self, device: &ash::Device, allocator: &vk_mem::Allocator) {
        let p = self.pipeline.take();
        if let Some(pip) = p {
            pip.cleanup(device);
        }
    }
    pub fn cleanup(&mut self, device: &ash::Device, allocator: &vk_mem::Allocator) {
        self.clear_pipeline(device, allocator);
        let p = self.desc_pool.take();
        if let Some(pool) = p {
            unsafe {
                device.destroy_descriptor_pool(pool, None);
            }
        }
        let b = self.vertexbuffer.take();
        if let Some(buf) = b {
            unsafe { allocator.destroy_buffer(buf.buffer, &buf.allocation) };
        }
        for texture in &self.textures {
            unsafe {
                device.destroy_sampler(texture.sampler, None);
                device.destroy_image_view(texture.imageview, None);
            }
            allocator.destroy_image(texture.vk_image, &texture.allocation);
        }
    }
```
While the pipeline should be cleared (and recreated, during `update_textures`) when the swapchain is renewed, it is not necessary to destroy all
textures. 


Also the textures from the last chapters have the same problem. 
```
[Debug][error][validation] "vkUpdateDescriptorSets() failed write update validation for VkDescriptorSet 0x2b000000002b[] with error: Cannot call vkUpdateDescriptorSets() to perfor
m write update on VkDescriptorSet VkDescriptorSet 0x2b000000002b[] allocated with VkDescriptorSetLayout VkDescriptorSetLayout 0x1d000000001d[] which has been destroyed. The Vulkan
 spec states: dstSet must be a valid VkDescriptorSet handle (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-VkWriteDescriptorSet-dstSet-00320)"
[Debug][error][validation] "descriptorSet #1 being bound is not compatible with overlapping descriptorSetLayout at index 1 of VkPipelineLayout 0x1020000000102[] due to: VkDescript
orSetLayout 0x1010000000101[] from pipeline layout has 1 total descriptors, but VkDescriptorSetLayout 0x1d000000001d[], which is bound, has 1048576 total descriptors.. The Vulkan 
spec states: Each element of pDescriptorSets must have been allocated with a VkDescriptorSetLayout that matches (is the same as, or identically defined as) the VkDescriptorSetLayo
ut at set n in layout, where n is the sum of firstSet and the index into pDescriptorSets (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-vkCmdB
indDescriptorSets-pDescriptorSets-00358)"
```
Actually, this discrepancy is due to a line (in `recreate_swapchain`) that should be 

```rust
        self.pipeline = Pipeline::init_textured(&self.device, &self.swapchain, &self.renderpass)?;
```
but was `Pipeline::init(&...)` (I had forgotten to adjust the pipeline here.) 

Nevertheless, still: 
```
 [Debug][error][validation] "vkUpdateDescriptorSets() failed write update validation for VkDescriptorSet 0x2b000000002b[] with error: Cannot call vkUpdateDescriptorSets() to perfor
m write update on VkDescriptorSet VkDescriptorSet 0x2b000000002b[] allocated with VkDescriptorSetLayout VkDescriptorSetLayout 0x1d000000001d[] which has been destroyed. The Vulkan
 spec states: dstSet must be a valid VkDescriptorSet handle (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VUID-VkWriteDescriptorSet-dstSet-00320)"
```
Right: The `pipeline::cleanup()` destroys descriptor sets. We do not notice it with our text, because there the descriptor sets are renewed during
`text::update_textures()` anyway.

Okay, another change to `recreate_swapchain`:
```rust
        self.pipeline = Pipeline::init_textured(&self.device, &self.swapchain, &self.renderpass)?;
        unsafe {
            self.device.reset_descriptor_pool(
                self.descriptor_pool,
                ash::vk::DescriptorPoolResetFlags::empty(),
            );
        }
        let desc_layouts_texture =
            vec![self.pipeline.descriptor_set_layouts[1]; self.swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info_texture = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(self.descriptor_pool)
            .set_layouts(&desc_layouts_texture);
        self.descriptor_sets_texture = unsafe {
            self.device
                .allocate_descriptor_sets(&descriptor_set_allocate_info_texture)
        }?;
        let desc_layouts_camera =
            vec![self.pipeline.descriptor_set_layouts[0]; self.swapchain.amount_of_images as usize];
        let descriptor_set_allocate_info_camera = vk::DescriptorSetAllocateInfo::builder()
            .descriptor_pool(self.descriptor_pool)
            .set_layouts(&desc_layouts_camera);
        self.descriptor_sets_camera = unsafe {
            self.device
                .allocate_descriptor_sets(&descriptor_set_allocate_info_camera)
        }?;

        for descset in &self.descriptor_sets_camera {
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: self.uniformbuffer.buffer,
                offset: 0,
                range: 128,
            }];
            let desc_sets_write = [vk::WriteDescriptorSet::builder()
                .dst_set(*descset)
                .dst_binding(0)
                .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
                .buffer_info(&buffer_infos)
                .build()];
            unsafe { self.device.update_descriptor_sets(&desc_sets_write, &[]) };
        }
```
(Most of this is copied from initialization.) New is the `self.device.reset_descriptor_pool` command. It, well, resets the descriptor pool, freeing
and recycling all descriptor sets in it. 

A new error: 
```
[Debug][error][validation] "vkUpdateDescriptorSets() failed write update validation for VkDescriptorSet 0x12f000000012f[] with error: Cannot call vkUpdateDescriptorSets() to perfo
rm write update on VkDescriptorSet VkDescriptorSet 0x12f000000012f[] allocated with VkDescriptorSetLayout VkDescriptorSetLayout 0x12b000000012b[] that is in use by a command buffe
r. The Vulkan spec states: All submitted commands that refer to any element of pDescriptorSets must have completed execution (https://www.khronos.org/registry/vulkan/specs/1.1-ext
ensions/html/vkspec.html#VUID-vkFreeDescriptorSets-pDescriptorSets-00309)"
```
What's that? 

Well, it turns out I have been using the wrong index when choosing which descriptor set to update. One-line fix:
```rust
            let descriptorwrite_image = vk::WriteDescriptorSet::builder()
                //.dst_set(aetna.descriptor_sets_texture[aetna.swapchain.current_image])
                .dst_set(aetna.descriptor_sets_texture[image_index as usize])
                .dst_binding(0)
                .dst_array_element(0)
                .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
                .image_info(&imageinfos)
                .build();
```


The overall design of `AllText` is not optimal: We can't remove letters yet. Also always having to call `create_letters`, `create_vertexdata`,
`update_vertexbuffer` after each other (by hand) is annoying. 

But rectifying that is something for another day (or, maybe, never). 

[Continue] ( When the next chapter exists. It may, again, take some time.)






