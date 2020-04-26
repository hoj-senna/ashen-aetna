## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Data for the vertex shader

We have seen how to pass data from vertex to fragment shader, but we don't want to put all data into the vertex shader, either. Let us try to send
some data to the vertex shader. Maybe the position first. 

According to the previous chapter, the vertex shader then should look like this: 
```glsl
#version 450

layout (location=0) in vec4 position;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_PointSize=10.0;
    gl_Position = position;
    colourdata_for_the_fragmentshader=vec4(0.4,1.0,0.5,1.0);
}
```

(And, just for the sake of completeness, the fragment shader still is 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec4 data_from_the_vertexshader;

void main(){
	theColour= data_from_the_vertexshader;
}
```
) 

Perhaps surprisingly, the program already runs. However, it does so with 
```
[Debug][error][validation] "Vertex shader consumes input at location 0 but not provided"
```

We have told the vertex shader that there will be input, but we have not told our Rust program about that yet. Somewhere we have to tell it that the
vertex shader now expects a `vec4` as argument, or, in rustier terms, a `[f32;4]`.

Of course, we cannot just write `layout (location=0) out` inside the Rust code and hope it to be understood — but there were many structures with some
information that we filled in. This "data for the vertex shader" is certainly related to the "pipeline". And indeed, in the long `Pipeline::init()`
function in our program ... 

```rust 
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
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder();
        let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::POINT_LIST);
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
        let pipelinelayout_info = vk::PipelineLayoutCreateInfo::builder();
        let pipelinelayout =
            unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
        let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&shader_stages)
            .vertex_input_state(&vertex_input_info)
            .input_assembly_state(&input_assembly_info)
            .viewport_state(&viewport_info)
            .rasterization_state(&rasterizer_info)
            .multisample_state(&multisampler_info)
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
        })
    }
```
... amidst many other entries, we find `let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder();`

Currently we are setting a default `PipelineVertexInputStateCreateInfo`, but this is easily changed: 
```rust
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
            .vertex_attribute_descriptions(&vertex_attrib_descs)
            .vertex_binding_descriptions(&vertex_binding_descs);
```
What we are setting here are attribute descriptions and binding descriptions. Compared to 
```glsl
layout (location=0) in vec4 position;
```
it is the attribute description that is carrying this information: 
```rust
        let vertex_attrib_descs = [vk::VertexInputAttributeDescription {
            binding: 0,
            location: 0,
            offset: 0,
            format: vk::Format::R32G32B32A32_SFLOAT,
        }];
```
`location` is easy to understand: This is the same as `location` in the shader. 

`format` is the variable type. And `R32G32B32A32_SFLOAT` is a strange way to spell `[f32;4]`. We recognize the "float" in both. Also the 32 for the
size of each component in bit is visible in both. There are four components: This is indicated by a number 4 in one case, and by letters and the
amount of repetition of "32" in the other case. Why the names R, G, B and A? Well, for colours they would make sense. These are no colours, but why
use different names? So R, G, B and A it is. If we wanted to pass some `[f32;2]`, we'd have to use `R32G32_SFLOAT` (and guess what to use for three or
one component!).

`offset`: Imagine, we had not one but two variables to pass, and we'd say to our pipeline: Here's the data. A reasonable question for the pipeline to
ask would be: Where do I find which part of the data, which variable. Offset answers this: How large is the offset (in bytes) from the beginning of
the data. For our variable: it's the first, so the offset is zero. If there were a second variable stored after it, that would get the offset 16 (four
variables of four bytes each is what we have put in the first place) — but as of now, there is no second variable. 

`binding`: There is not yet any visible reason for the entry "binding". But imagine we had to completely different things we wanted to send to the
shaders. Say, position and colour, where we want to be able to change the colour data very often, but the positions stay the same for long times.
Then it could be good if we could update the colour data independently of the positions. Then it's beneficial to have these different types of
information in different places, have them "bound" to different ... well, bindings.

Of course, these bindings must be described: 
```rust
        let vertex_binding_descs = [vk::VertexInputBindingDescription {
            binding: 0,
            stride: 16,
            input_rate: vk::VertexInputRate::VERTEX,
        }];
```
We specify, which binding it is, and how large all variables at this binding together are (16 bytes in our case, see above), that is, how large the
step from the set of data for one vertex to that of the next vertex is. And then there is an `input rate`, where we say that the data at this binding
changes from one vertex to the next. The other option would be `INSTANCE` and we'll have to talk about that when we deal with instanced rendering
(later). 

So, with these lines 
```rust
        let vertex_attrib_descs = [vk::VertexInputAttributeDescription {
            binding: 0,
            location: 0,
            offset: 0,
            format: vk::Format::R32G32B32A32_SFLOAT,
        }];
        let vertex_binding_descs = [vk::VertexInputBindingDescription {
            binding: 0,
            stride: 16,
            input_rate: vk::VertexInputRate::VERTEX,
        }];
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
            .vertex_attribute_descriptions(&vertex_attrib_descs)
            .vertex_binding_descriptions(&vertex_binding_descs);
```
we have described what data we want to pass to our pipeline. We have not yet given any actual data to it.

Now, data is never just given to the GPU as some "here you are, these are the variables". Instead we first create some "buffer" and fill in all our
variables, and then we tell Vulkan: Here, the data is in this buffer.

That means, we have to setup a buffer. And actually, we have to reserve suitable memory for it separately, as well. Vulkan gives us control over
memory handling — and expects us to deal with it. Also it is recommended (though not yet important on the stage of tutorial code) not to allocate
every buffer separately, but to allocate one large buffer and then use parts of it. And with the allocation, certain alignment requirements should be
satisfied, ... 

We will try to make this whole topic a little easier on us and not use Vulkan directly, but the Vulkan Memory Allocator, VMA. A new line goes into
Cargo.toml: `vk-mem = "0.2.2"`

We first create an allocator: 

```rust
        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let mut allocator = vk_mem::Allocator::new(&allocator_create_info)?;
```
Like everything with Vulkan, this comes with some `CreateInfo` struct that we have to fill in first. `vk_mem` does not subscribe to the nice
`::builder()` functionality that ash uses, so we fall back to a `..Default::default()` variant, together with some explicit list. (And for some
reason, we have to clone device and instance (because it's ash::Instance and not ash::vk::Instance which would implement `Copy`.)

With this allocator, we then create a buffer: 
```rust
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(16)
                .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
                .build(),
            &allocation_create_info,
        )?;
```
As you can see, there is some `BufferCreateInfo` involved, and we set the size of the buffer (in bytes) and what we want to use it for. In this case,
as a vertex buffer. And, apparently, we need some `AllocationCreateInfo`. Here it is: 
```rust
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::CpuToGpu,
            ..Default::default()
        };
```
What we describe here is what [type of memory](https://docs.rs/vk-mem/0.2.2/vk_mem/enum.MemoryUsage.html) we want. CpuOnly/GpuOnly/CpuToGpu/GpuToCpu?
Here we choose CpuToGpu: We want to write to this buffer directly on the CPU, and we want to access it from the GPU. 

We then fill the buffer with data: 
```rust
        let data_ptr = allocator.map_memory(&allocation)? as *mut f32;
        let data = [-0.5f32, 0.0f32, 0.0f32, 1.0f32];
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), 4) };
        allocator.unmap_memory(&allocation);
```
We first `map` the memory so that we can access it. Then we invent some data (the coordinates we want to pass to the shader!), copy it, and finally
`unmap` the memory. (`unmap` has to be called as often as `map`, but this doesn't have to happen as fast as here.) 

Now that we have filled the buffer, we still have to use it. We include one more line in the recording of the command buffer: 
```rust
       unsafe {
            logical_device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            logical_device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                pipeline.pipeline,
            );
            logical_device.cmd_bind_vertex_buffers(commandbuffer, 0, &[*vb], &[0]);
            logical_device.cmd_draw(commandbuffer, 1, 1, 0, 0);
            logical_device.cmd_end_render_pass(commandbuffer);
            logical_device.end_command_buffer(commandbuffer)?;
        }
```
The line 
```rust
            logical_device.cmd_bind_vertex_buffers(commandbuffer, 0, &[*vb], &[0]);
```
is the new line, and `vb` is a reference to `buffer`. The first `0` among the function arguments is the binding to be updated by this buffer. (I
mentioned bindings above.) If we wanted to update several bindings at the same time, that's possible, that's why there is a slice of buffers involved.

The last argument indicates offsets, in case we do not want to use all entries of the vertexbuffer(s) provided.

One more thing remains to do: destruction of buffer and allocator: 
```rust
            self.allocator.destroy_buffer(self.buffer, &self.allocation);
            self.allocator.destroy();
```

We have changed our `Aetna` struct, its `init` and `drop` functions, and the `fill_commandbuffer` function. Here they are: 
```rust
fn fill_commandbuffers(
    commandbuffers: &[vk::CommandBuffer],
    logical_device: &ash::Device,
    renderpass: &vk::RenderPass,
    swapchain: &SwapchainDongXi,
    pipeline: &Pipeline,
    vb: &vk::Buffer,
) -> Result<(), vk::Result> {
    for (i, &commandbuffer) in commandbuffers.iter().enumerate() {
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            logical_device.begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [vk::ClearValue {
            color: vk::ClearColorValue {
                float32: [0.0, 0.0, 0.08, 1.0],
            },
        }];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(*renderpass)
            .framebuffer(swapchain.framebuffers[i])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: swapchain.extent,
            })
            .clear_values(&clearvalues);
       unsafe {
            logical_device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            logical_device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                pipeline.pipeline,
            );
            logical_device.cmd_bind_vertex_buffers(commandbuffer, 0, &[*vb], &[0]);
            logical_device.cmd_draw(commandbuffer, 1, 1, 0, 0);
            logical_device.cmd_end_render_pass(commandbuffer);
            logical_device.end_command_buffer(commandbuffer)?;
        }
    }
    Ok(())
}

struct Aetna {
    window: winit::window::Window,
    entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    physical_device: vk::PhysicalDevice,
    physical_device_properties: vk::PhysicalDeviceProperties,
    queue_families: QueueFamilies,
    queues: Queues,
    device: ash::Device,
    swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    commandbuffers: Vec<vk::CommandBuffer>,
    allocator: vk_mem::Allocator,
    buffer: vk::Buffer,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
}

impl Aetna {
    fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;
        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &queues,
        )?;
        let renderpass = init_renderpass(
            &logical_device,
            physical_device,
            swapchain.surface_format.format,
        )?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;

        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let mut allocator = vk_mem::Allocator::new(&allocator_create_info)?;
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::CpuToGpu,
            ..Default::default()
        };
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(16)
                .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
                .build(),
            &allocation_create_info,
        )?;
        let data_ptr = allocator.map_memory(&allocation)? as *mut f32;
        let data = [-0.5f32, 0.0f32, 0.0f32, 1.0f32];
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), 4) };
        allocator.unmap_memory(&allocation);

        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;
        fill_commandbuffers(
            &commandbuffers,
            &logical_device,
            &renderpass,
            &swapchain,
            &pipeline,
            &buffer,
        )?;

        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            buffer,
            allocation,
            allocation_info,
        })
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.allocator.destroy_buffer(self.buffer, &self.allocation);
            self.allocator.destroy();
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device);
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
```

And now we can run the program. And changing the values in `let data = [-0.5f32, 0.0f32, 0.0f32, 1.0f32];` changes the coordinates of the point we are
drawing. 

So, this is how we get data to the vertex shader. Great! 

Let's send *more* data to it. 

There were more variables than just the position: We also had colour and pointsize. 
```glsl
#version 450

layout (location=0) in vec4 position;
layout (location=1) in float size;
layout (location=2) in vec4 colour;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_PointSize=size;
    gl_Position = position;
    colourdata_for_the_fragmentshader=colour;
}
```

Of course, we now should pass accordingly more floats into the buffer (which now has to be larger):
```rust
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(36)
                .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
                .build(),
            &allocation_create_info,
        )?;
        let data_ptr = allocator.map_memory(&allocation)? as *mut f32;
        let data = [
            0.1f32, -0.3f32, 0.0f32, 1.0f32, 5.0f32, 1.0f32, 1.0f32, 0.0f32, 1.0f32,
        ];
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), 9) };
```
But wait! We also have to adjust the attribute descriptions: 
```rust
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                offset: 16,
                format: vk::Format::R32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 2,
                offset: 20,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
        ];
```

All for binding 0, those entries for locations 1 and 2 are new, corresponding to the new input variables in the shader. The point size is only one
float `Format::R32_SFLOAT`, colour has four 32-bit components (and here the name `R32G32B32A32` is entirely appropriate). The offsets are easily
computed by simple counting and remembering that each `f32` float is 4 bytes large.

Run the program, and play a bit with the values in `data`. 



Next, let us put these in different buffers. Let's say, the position data in one, and colour and size in the other. 

Okay, then let's create two buffers: 
```rust
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::CpuToGpu,
            ..Default::default()
        };
        let (buffer1, allocation1, allocation_info1) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(16)
                .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
                .build(),
            &allocation_create_info,
        )?;
        let data_ptr = allocator.map_memory(&allocation1)? as *mut f32;
        let data = [0.1f32, -0.3f32, 0.0f32, 1.0f32];
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), 4) };
        allocator.unmap_memory(&allocation1);
        let (buffer2, allocation2, allocation_info2) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(20)
                .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
                .build(),
            &allocation_create_info,
        )?;
        let data_ptr = allocator.map_memory(&allocation2)? as *mut f32;
        let data = [5.0f32, 1.0f32, 1.0f32, 0.0f32, 1.0f32];
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), 5) };
        allocator.unmap_memory(&allocation2);
```
(I have just copied the buffer creation part, and fill in the same values as before, just some in the first, some in the second buffer.)

And this has changed our main pass-everything-around struct: 
```rust
        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            buffer1,
            allocation1,
            allocation_info1,
            buffer2,
            allocation2,
            allocation_info2,
        })
```
together with 
```rust
struct Aetna {
    window: winit::window::Window,
    entry: ash::Entry,
    instance: ash::Instance,
    debug: std::mem::ManuallyDrop<DebugDongXi>,
    surfaces: std::mem::ManuallyDrop<SurfaceDongXi>,
    physical_device: vk::PhysicalDevice,
    physical_device_properties: vk::PhysicalDeviceProperties,
    queue_families: QueueFamilies,
    queues: Queues,
    device: ash::Device,
    swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    commandbuffers: Vec<vk::CommandBuffer>,
    allocator: vk_mem::Allocator,
    buffer1: vk::Buffer,
    allocation1: vk_mem::Allocation,
    allocation_info1: vk_mem::AllocationInfo,
    buffer2: vk::Buffer,
    allocation2: vk_mem::Allocation,
    allocation_info2: vk_mem::AllocationInfo,
}
```
and 
```rust
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.allocator
                .destroy_buffer(self.buffer1, &self.allocation1);
            self.allocator
                .destroy_buffer(self.buffer2, &self.allocation2);
```

Important to adjust are the descriptions: 
```rust
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 1,
                offset: 0,
                format: vk::Format::R32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 2,
                offset: 4,
                format: vk::Format::R32G32B32A32_SFLOAT,
            },
        ];
```
We have shifted second and third attribute to `binding: 1`. That means, their offset had to be recomputed: It starts from the beginning of each data
element in the corresponding buffer.

The `location`, however, is not changed. All of the vertex shader input bindings share one namespace. (That also means that we don't have to change
the vertex shader for this change.) 

Before we move on, a bit of code cleanup. I don't like having six new member variables of `Aetna` just for two buffers. 

Let's introduce a separate Buffer struct which carries the buffer and its `vk_mem`-related information:
```rust
struct Buffer {
    buffer: vk::Buffer,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
}
```
We extract the whole creation into a `new` function: 
```rust
impl Buffer {
    fn new(
        allocator: &vk_mem::Allocator,
        size_in_bytes: u64,
        usage: vk::BufferUsageFlags,
        memory_usage: vk_mem::MemoryUsage,
    ) -> Result<Buffer, vk_mem::error::Error> {
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: memory_usage,
            ..Default::default()
        };
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(size_in_bytes)
                .usage(usage)
                .build(),
            &allocation_create_info,
        )?;
        Ok(Buffer {
            buffer,
            allocation,
            allocation_info,
        })
    }
}
```
and also filling the buffer becomes a function of its own. We make it generic over the type of data: 

```rust
    fn fill<T: Sized>(
        &self,
        allocator: &vk_mem::Allocator,
        data: &[T],
    ) -> Result<(), vk_mem::error::Error> {
        let data_ptr = allocator.map_memory(&self.allocation)? as *mut T;
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), data.len()) };
        allocator.unmap_memory(&self.allocation);
        Ok(())
    }
```

We then put a `vec![buffer1,buffer2]` into `Aetna` after creating and filling these two buffers. A rather complete chunk of code of this section: 

```rust
fn fill_commandbuffers(
    commandbuffers: &[vk::CommandBuffer],
    logical_device: &ash::Device,
    renderpass: &vk::RenderPass,
    swapchain: &SwapchainDongXi,
    pipeline: &Pipeline,
    vb1: &vk::Buffer,
    vb2: &vk::Buffer,
) -> Result<(), vk::Result> {
    for (i, &commandbuffer) in commandbuffers.iter().enumerate() {
        let commandbuffer_begininfo = vk::CommandBufferBeginInfo::builder();
        unsafe {
            logical_device.begin_command_buffer(commandbuffer, &commandbuffer_begininfo)?;
        }
        let clearvalues = [vk::ClearValue {
            color: vk::ClearColorValue {
                float32: [0.0, 0.0, 0.08, 1.0],
            },
        }];
        let renderpass_begininfo = vk::RenderPassBeginInfo::builder()
            .render_pass(*renderpass)
            .framebuffer(swapchain.framebuffers[i])
            .render_area(vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: swapchain.extent,
            })
            .clear_values(&clearvalues);
        unsafe {
            logical_device.cmd_begin_render_pass(
                commandbuffer,
                &renderpass_begininfo,
                vk::SubpassContents::INLINE,
            );
            logical_device.cmd_bind_pipeline(
                commandbuffer,
                vk::PipelineBindPoint::GRAPHICS,
                pipeline.pipeline,
            );
            logical_device.cmd_bind_vertex_buffers(commandbuffer, 0, &[*vb1], &[0]);
            logical_device.cmd_bind_vertex_buffers(commandbuffer, 1, &[*vb2], &[0]);
            logical_device.cmd_draw(commandbuffer, 1, 1, 0, 0);
            logical_device.cmd_end_render_pass(commandbuffer);
            logical_device.end_command_buffer(commandbuffer)?;
        }
    }
    Ok(())
}

struct Buffer {
    buffer: vk::Buffer,
    allocation: vk_mem::Allocation,
    allocation_info: vk_mem::AllocationInfo,
}

impl Buffer {
    fn new(
        allocator: &vk_mem::Allocator,
        size_in_bytes: u64,
        usage: vk::BufferUsageFlags,
        memory_usage: vk_mem::MemoryUsage,
    ) -> Result<Buffer, vk_mem::error::Error> {
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: memory_usage,
            ..Default::default()
        };
        let (buffer, allocation, allocation_info) = allocator.create_buffer(
            &ash::vk::BufferCreateInfo::builder()
                .size(size_in_bytes)
                .usage(usage)
                .build(),
            &allocation_create_info,
        )?;
        Ok(Buffer {
            buffer,
            allocation,
            allocation_info,
        })
    }
    fn fill<T: Sized>(
        &self,
        allocator: &vk_mem::Allocator,
        data: &[T],
    ) -> Result<(), vk_mem::error::Error> {
        let data_ptr = allocator.map_memory(&self.allocation)? as *mut T;
        unsafe { data_ptr.copy_from_nonoverlapping(data.as_ptr(), data.len()) };
        allocator.unmap_memory(&self.allocation)?;
        Ok(())
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
    queue_families: QueueFamilies,
    queues: Queues,
    device: ash::Device,
    swapchain: SwapchainDongXi,
    renderpass: vk::RenderPass,
    pipeline: Pipeline,
    pools: Pools,
    commandbuffers: Vec<vk::CommandBuffer>,
    allocator: vk_mem::Allocator,
    buffers: Vec<Buffer>,
}

impl Aetna {
    fn init(window: winit::window::Window) -> Result<Aetna, Box<dyn std::error::Error>> {
        let entry = ash::Entry::new()?;

        let layer_names = vec!["VK_LAYER_KHRONOS_validation"];
        let instance = init_instance(&entry, &layer_names)?;
        let debug = DebugDongXi::init(&entry, &instance)?;
        let surfaces = SurfaceDongXi::init(&window, &entry, &instance)?;

        let (physical_device, physical_device_properties) =
            init_physical_device_and_properties(&instance)?;

        let queue_families = QueueFamilies::init(&instance, physical_device, &surfaces)?;

        let (logical_device, queues) =
            init_device_and_queues(&instance, physical_device, &queue_families, &layer_names)?;
        let mut swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &queues,
        )?;
        let renderpass = init_renderpass(
            &logical_device,
            physical_device,
            swapchain.surface_format.format,
        )?;
        swapchain.create_framebuffers(&logical_device, renderpass)?;
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;
        let pools = Pools::init(&logical_device, &queue_families)?;

        let allocator_create_info = vk_mem::AllocatorCreateInfo {
            physical_device,
            device: logical_device.clone(),
            instance: instance.clone(),
            ..Default::default()
        };
        let allocator = vk_mem::Allocator::new(&allocator_create_info)?;
        let allocation_create_info = vk_mem::AllocationCreateInfo {
            usage: vk_mem::MemoryUsage::CpuToGpu,
            ..Default::default()
        };
        let buffer1 = Buffer::new(
            &allocator,
            16,
            vk::BufferUsageFlags::VERTEX_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer1.fill(&allocator, &[0.4f32, -0.2f32, 0.0f32, 1.0f32])?;
        let buffer2 = Buffer::new(
            &allocator,
            20,
            vk::BufferUsageFlags::VERTEX_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer2.fill(&allocator, &[15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32])?;

        let commandbuffers =
            create_commandbuffers(&logical_device, &pools, swapchain.amount_of_images)?;
        fill_commandbuffers(
            &commandbuffers,
            &logical_device,
            &renderpass,
            &swapchain,
            &pipeline,
            &buffer1.buffer,
            &buffer2.buffer,
        )?;

        Ok(Aetna {
            window,
            entry,
            instance,
            debug: std::mem::ManuallyDrop::new(debug),
            surfaces: std::mem::ManuallyDrop::new(surfaces),
            physical_device,
            physical_device_properties,
            queue_families,
            queues,
            device: logical_device,
            swapchain,
            renderpass,
            pipeline,
            pools,
            commandbuffers,
            allocator,
            buffers: vec![buffer1, buffer2],
        })
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            for b in &self.buffers {
                self.allocator
                    .destroy_buffer(b.buffer, &b.allocation)
                    .expect("problem with buffer destruction");
            }
            self.allocator.destroy();
            self.pools.cleanup(&self.device);
            self.pipeline.cleanup(&self.device);
            self.device.destroy_render_pass(self.renderpass, None);
            self.swapchain.cleanup(&self.device);
            self.device.destroy_device(None);
            std::mem::ManuallyDrop::drop(&mut self.surfaces);
            std::mem::ManuallyDrop::drop(&mut self.debug);
            self.instance.destroy_instance(None)
        };
    }
}
```


[Continue](019_More_points.md)
