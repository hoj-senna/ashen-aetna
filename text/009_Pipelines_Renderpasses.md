## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Pipelines, Renderpasses 

In order to draw on the screen (or rather on the images provided by the swapchain, see above), several steps will be taken, some of them configurable.
Together, they are referred to as "the graphics pipeline". We will try for a rather minimal version now, and will first look at the last step. The
last step is to actually apply some colour. We line up servants with paint and brush (metaphorically, of course) and tell them to start painting.
Actually, we employ one painter per pixel (and are very happy that they are not actual painters and that our computer works at a very cheap salary).

What does each painter have to know? 

Obviously: Which colour to use. We write some instructions for them. (Unusually, not yet meant for direct inclusion in our program code. We'll save
the following lines in a separate file 'shader.frag' in the new folder 'shaders' to be created as subfolder of the folder containing Cargo.toml.) 
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	void main(){
		theColour= vec4(1.0,0.0,0.0,1.0);
	}
```
This is a shader program for the fragment shader (which is the name of our painters). Vulkan expects shader code to be supplied in a compiled format
(SPIRV), but we can write it in GLSL (the first line of the program specifying the language version, here 4.50) and compile it before using. 

What are we doing in this shader? In the main function we set some variable (which we will interpret as colour) to red, red consisting of the four
components 1 (red), 0 (green), 0 (blue), 1 (alpha, i.e.: non-transparent) which are provided in a [0,1] range. We also have to state that this
variable is the output of our shader ("`out vec4 theColour`") and where to put it (the "`location=0`" bit, which we will talk about a bit later). 
Of course, we could make these instructions much more complicated than setting it to red only. There is a whole programming language at our disposal,
after all (even though some limitations do apply) and we can also pass some data to this program (a topic for later). We cannot, however, give each of
the painters (=each fragment=each pixel) a different program. 

But we can choose — or more correctly: let the GPU choose — which of the painters go to work and follow their instructions, and which of the painters
stay idle. ("Sorry, Vincent, nothing to do for you today".) How does it choose? Well, everyone gets to paint who has something in his pixel.

"Something"? Some thing, some geometry. Suppose, there is a point at (0.5,0.2,0.7) (in some coordinate system that we will also have to talk about).

Then this point corresponds to a pixel on the screen, and the fragment shader at that point may paint. How do we now at which points there is
something? That is what the vertex shader tells us. If the vertex shader says: At this point we have something, the fragment shader there is invoked.

Actually, this depends a bit on the setup of the pipeline: It might as well be that for every three points the vertex shader spits out, the fragment
shaders at all pixels corresponding to a triangle spanned by those points are run. A vertex shader could be the following: 
```glsl
	#version 450

	void main() {
	    gl_Position = vec4(0.0,0.0,0.0,1.0);
	}
```
The output goes to a special variable, `gl_Position`. Its meaning is rather fixed, it is what I have described before: Position data needed to figure
out which pixels have to be drawn. Again it is a vec4, four components. Four? For 3D graphics, I'd understand 3 components. Think of the fourth
component "1.0" as a marker "this is a point" (as opposed to "this is a direction"); we will talk about the meaning later. 

Mean question: How large is a triangle between the three points (0,0,0), (0,0,0) and (0,0,0)? Nothing to see here. This vertex shader is a bit too
simple. We can postpone our goalt to draw a triangle and settle for a single point. How big is a point? I don't know, you tell me. In this case that
means we include a second prebuilt output variable: 
```glsl 
	#version 450

	void main() {
	    gl_PointSize=2.0;
	    gl_Position = vec4(0.0,0.0,0.0,1.0);
	}
```
(And we're saving this as 'shader.vert' in the shader folder.) 

What we would do in a real program is passing some data (like the position itself) in from the application. (By now you have seen enough of Vulkan
that you can guess this means a lot more setup, which I'd like to avoid for the moment.) Another possible solution is to store several positions in
the vertex shader and to emit "this one the first time the shader is called", "this one the second time" and "this one on the third invocation". If
you're interested in the shader (close to the example at vulkan-tutorial.com): 
```glsl
	#version 450

	vec4 positions[3] = vec4[](
	    vec4(0.0, -0.5,0.0,1.0),
	    vec4(0.5, 0.5,0.0,1.0),
	    vec4(-0.5, 0.5,0.0,1.0)
	);

	void main() {
	    gl_Position = positions[gl_VertexIndex];
	}
```
The vertex shader is the thing that is fed with data from the application. Short pipeline? Yes, some stages are optional (and possibly something to
look at later). And some stages are fixed. There was this "find out which fragments are affected, given the positions output by the vertex shader". 
That's a "rasterizer stage" right there. 

Even in this very simple example we have touched upon some more knobs to twist (for example, the question of whether to interpret the input as single
points or three together as one triangle), and, of course, there are some Vulkan objects to create (and corresponding CreateInfo structs to fill). One
of these objects is a `vk::Pipeline` (no surprise there). 

Pipelines are, once created, never changed, but there can be many of them. The second thing of interest is a renderpass. In a somewhat abstracted view, 
what we have seen above was the process "some image is (by whatever painting method) turned into some other image" and we have even (in the fragment shader,
 "`layout (location=0) out ...`") once had a hint that there might be several different images involved. 

The abstracted description which images (the better, technical term here would be "attachments") are used for, generally speaking,
their purpose, is encoded in a renderpass, which in turn can consist of several subpasses. For now, there is one subpass and a single attachment (used
as colour). Attachment first [all code modifications taking place between "`swapchain=...`" and "`Ok(Aetna{...})`"]:  
```rust
        let attachments = [vk::AttachmentDescription::builder()
            .format(
                surfaces
                    .get_formats(physical_device)?
                    .first()
                    .unwrap()
                    .format,
            )
            .load_op(vk::AttachmentLoadOp::CLEAR)
            .store_op(vk::AttachmentStoreOp::STORE)
            .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
            .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
            .samples(vk::SampleCountFlags::TYPE_1)
            .build()];
```
(There may be several, so we need an array of them, and therefore also have to `.build()`, like earlier.)
The format must be the same as for the swapchain. we then have to decide what to do in the beginning. keep any previous content? clear it? (or:
doesn't matter, because we either do not intend to use it or we will write on every pixel anyway). we clear the colour. a similar decision has to be
made about what shall happen at the end of the render pass. throw the content away? here, we want to keep (store) it; after all, we want to show it on
screen. And we must decide on the way the data is organized in memory at beginning and end of the pass (initial and final layout). We don't know at
the beginning, and at the end it should be such that we can present it in our swapchain. The final option is how many samples per pixel there should
be. (If for antialiasing/smoothing effects one pixel is not small enough, let's pretend to have different colours for smaller parts of pixels.) No,
thanks, not now. During rendering, we will reference this attachment as attachment no 0 (cf. shader earlier): 
```rust
        let color_attachment_references = [vk::AttachmentReference {
            attachment: 0,
            layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
        }];
```
(We also say how (optimized for what purpose) to lay out the data in memory during the pass.) This is enough information to get a
subpass(description), which, just by the way, is made for graphics pipelines, not for compute pipelines: 
```rust
        let subpasses = [vk::SubpassDescription::builder()
            .color_attachments(&color_attachment_references)
            .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS).build()];
```
There could be several subpasses. In that case, Vulkan would like to optimize (execution order for commands), and would like to know how these
subpasses depend on each other: Which part of subpass A has to be finished for (wich part of) subpass B to do its work? Now, we only have one subpass,
but there is still a dependency: Before we can start painting, we must have something to paint on. We can include this in the following way: When the
preparations ("`SUBPASS_EXTERNAL`", not an actual subpass) have finished their work on "`COLOR_ATTACHMENT_OUTPUT`" (have prepared the attachment and
transformed it in the right layout), our subpass (at index 0 in the array we are preparing) may begin its own output to colour attachments, and only
then may read from or write to the colour attachment:
```rust
        let subpass_dependencies = [vk::SubpassDependency::builder()
            .src_subpass(vk::SUBPASS_EXTERNAL)
            .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
            .dst_subpass(0)
            .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
            .dst_access_mask(
                vk::AccessFlags::COLOR_ATTACHMENT_READ | vk::AccessFlags::COLOR_ATTACHMENT_WRITE,
            )
            .build()];
```
And nothing left to explain here: 
```rust
        let renderpass_info = vk::RenderPassCreateInfo::builder()
            .attachments(&attachments)
            .subpasses(&subpasses)
            .dependencies(&subpass_dependencies);
        let renderpass = unsafe { logical_device.create_render_pass(&renderpass_info, None)? };
        logical_device.destroy_render_pass(renderpass, None);
```
Of course, we should move all of this to its own function and extend Aetna by another field. 
```rust
fn init_renderpass(
    logical_device: &ash::Device,
    physical_device: vk::PhysicalDevice,
    surfaces: &SurfaceDongXi,
) -> Result<vk::RenderPass, vk::Result> {
    let attachments = [vk::AttachmentDescription::builder()
        .format(
            surfaces
                .get_formats(physical_device)?
                .first()
                .unwrap()
                .format,
        )
        .load_op(vk::AttachmentLoadOp::CLEAR)
        .store_op(vk::AttachmentStoreOp::STORE)
        .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
        .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
        .initial_layout(vk::ImageLayout::UNDEFINED)
        .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
        .samples(vk::SampleCountFlags::TYPE_1)
        .build()];
    let color_attachment_references = [vk::AttachmentReference {
        attachment: 0,
        layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
    }];
    let subpasses = [vk::SubpassDescription::builder()
        .color_attachments(&color_attachment_references)
        .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
        .build()];
    let subpass_dependencies = [vk::SubpassDependency::builder()
        .src_subpass(vk::SUBPASS_EXTERNAL)
        .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_subpass(0)
        .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_access_mask(
            vk::AccessFlags::COLOR_ATTACHMENT_READ | vk::AccessFlags::COLOR_ATTACHMENT_WRITE,
        )
        .build()];
    let renderpass_info = vk::RenderPassCreateInfo::builder()
        .attachments(&attachments)
        .subpasses(&subpasses)
        .dependencies(&subpass_dependencies);
    let renderpass = unsafe { logical_device.create_render_pass(&renderpass_info, None)? };
    Ok(renderpass)
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
        let swapchain = SwapchainDongXi::init(
            &instance,
            physical_device,
            &logical_device,
            &surfaces,
            &queue_families,
            &queues,
        )?;
        let renderpass = init_renderpass(&logical_device, physical_device, &surfaces)?;

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
        })
    }
}

impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
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

During the previous paragraphs about the renderpass, we have dealt with the "attachments" that were the "images" from the swapchain (and my words
meandered between descriptive and technically appropriate). We did have ImageViews from the swapchain. We somehow have to couple them with the
"attachments" of the renderpass. The Framebuffer is the concept bringing both together: We create some as part of our SwapchainDongXi struct: 
```rust
    fn create_framebuffers(
        &mut self,
        logical_device: &ash::Device,
        renderpass: vk::RenderPass,
    ) -> Result<(), vk::Result> {
        for iv in &self.imageviews {
            let iview = [*iv];
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
The information given is the renderpass, the imageview, and the size of the swapchain (which I also save in the struct while I'm at it).
A copy of all affected code (plus a bit more): 
```rust
struct SwapchainDongXi {
    swapchain_loader: ash::extensions::khr::Swapchain,
    swapchain: vk::SwapchainKHR,
    images: Vec<vk::Image>,
    imageviews: Vec<vk::ImageView>,
    framebuffers: Vec<vk::Framebuffer>,
    surface_format: vk::SurfaceFormatKHR,
    extent: vk::Extent2D,
}

impl SwapchainDongXi {
    fn init(
        instance: &ash::Instance,
        physical_device: vk::PhysicalDevice,
        logical_device: &ash::Device,
        surfaces: &SurfaceDongXi,
        queue_families: &QueueFamilies,
        queues: &Queues,
    ) -> Result<SwapchainDongXi, vk::Result> {
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
        Ok(SwapchainDongXi {
            swapchain_loader,
            swapchain,
            images: swapchain_images,
            imageviews: swapchain_imageviews,
            framebuffers: vec![],
            surface_format,
            extent,
        })
    }
    fn create_framebuffers(
        &mut self,
        logical_device: &ash::Device,
        renderpass: vk::RenderPass,
    ) -> Result<(), vk::Result> {
        for iv in &self.imageviews {
            let iview = [*iv];
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
    unsafe fn cleanup(&mut self, logical_device: &ash::Device) {
        for fb in &self.framebuffers {
            logical_device.destroy_framebuffer(*fb, None);
        }
        for iv in &self.imageviews {
            logical_device.destroy_image_view(*iv, None);
        }
        self.swapchain_loader
            .destroy_swapchain(self.swapchain, None)
    }
}

fn init_renderpass(
    logical_device: &ash::Device,
    physical_device: vk::PhysicalDevice,
    format: vk::Format,
) -> Result<vk::RenderPass, vk::Result> {
    let attachments = [vk::AttachmentDescription::builder()
        .format(format)
        .load_op(vk::AttachmentLoadOp::CLEAR)
        .store_op(vk::AttachmentStoreOp::STORE)
        .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
        .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
        .initial_layout(vk::ImageLayout::UNDEFINED)
        .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
        .samples(vk::SampleCountFlags::TYPE_1)
        .build()];
    let color_attachment_references = [vk::AttachmentReference {
        attachment: 0,
        layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
    }];
    let subpasses = [vk::SubpassDescription::builder()
        .color_attachments(&color_attachment_references)
        .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
        .build()];
    let subpass_dependencies = [vk::SubpassDependency::builder()
        .src_subpass(vk::SUBPASS_EXTERNAL)
        .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_subpass(0)
        .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
        .dst_access_mask(
            vk::AccessFlags::COLOR_ATTACHMENT_READ | vk::AccessFlags::COLOR_ATTACHMENT_WRITE,
        )
        .build()];
    let renderpass_info = vk::RenderPassCreateInfo::builder()
        .attachments(&attachments)
        .subpasses(&subpasses)
        .dependencies(&subpass_dependencies);
    let renderpass = unsafe { logical_device.create_render_pass(&renderpass_info, None)? };
    Ok(renderpass)
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
        })
    }
}
```

Oookay. Time to create the pipeline. We have saved our shaders as separate files, as glsl code. Vulkan expects SPIRV, so we could (by using glslc)
compile these files into .spv files and load those. Or we could have that be done automatically and include the files with GLSL code directly. I find
the latter more convenient. We use another crate for that: 
```
	vk-shader-macros = "0.2.2"
```
is the new line for Cargo.toml; and then we create a vertex shader module the following way: 
```rust
        let vertexshader_createinfo = vk::ShaderModuleCreateInfo::builder().code(
            vk_shader_macros::include_glsl!("./shaders/shader.vert", kind: vert),
        );
        let vertexshader_module =
            unsafe { logical_device.create_shader_module(&vertexshader_createinfo, None)? };
```
In the `include_glsl!`-macro, the "`kind: vert`" information is redundant; it's only necessary if we should use a different file extension. For the
fragment shader: 
```rust
        let fragmentshader_createinfo = vk::ShaderModuleCreateInfo::builder()
            .code(vk_shader_macros::include_glsl!("./shaders/shader.frag"));
        let fragmentshader_module =
            unsafe { logical_device.create_shader_module(&fragmentshader_createinfo, None)? };
```
We can decide which function in the shader to use as entry point (we called it "main" in both cases and have no reason to use anything else at the
moment) and then turn the shader modules into shader stages, the things to include in the pipeline: 
```rust
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
```
There is, however, quite a bit more to decide on, before Vulkan knows everything about the pipeline that there is to know. The first thing is: What
kind of data do we pass from our application to the shaders, or rather: to the vertex shader? 
```rust
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder();
```
Nothing, everything we needed came from the shader itself. Therefore (for now, in this untypical situation), no methods after `::builder()`. Are we
going to pass in single points or triangles or ...? Well, points for now. (If we chose triangles, we'd have to call the vertex shader thrice to get
anything at all, and that seems a waste, where we have it set to output the same point everytime...)
```rust
         let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::POINT_LIST);
```
Which part of the screen do we want to correspond to Vulkan's internal coordinates (viewport) and can we permanently disable drawing outside a certain
arrea (scissors)? 
```rust
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
```
How do we translate the points (output of the vertex shader) into a decision which fragments are painted? For points this may not be that interesting,
so let's for the moment assume, we have a triangle. Should we paint it as a filled triangle? Or only as the lines (as a wireframe)? Or only the
points? (`.polygon_mode(...)`) 

Speaking of lines: How thick are they? 

Should we ignore ("cull") certain triangles? - I'm choosing no, now. But ignoring all triangles that we would only see the backside of could be a good
idea (and save some work). If we want that, we have to explain what front and back are. E.g.: We can see the triangle ABC if A, B and C are in
counterclockwise order as seen from the point of the camera. (Choosing the wrong option here, can easily produce the classical graphics error of
"everything should work, but I see nothing at all".) Here we go: 
```rust
        let rasterizer_info = vk::PipelineRasterizationStateCreateInfo::builder()
            .line_width(1.0)
            .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
            .cull_mode(vk::CullModeFlags::NONE)
            .polygon_mode(vk::PolygonMode::FILL);
```
Multisampling? We said something about that earlier, when creating the renderpass. No, only one sample per pixel: 
```rust
        let multisampler_info = vk::PipelineMultisampleStateCreateInfo::builder()
            .rasterization_samples(vk::SampleCountFlags::TYPE_1);
```
We also can choose how to deal with transparence, that is: with the alpha value, the fourth value describing a colour. If our "painter" is supposed to
put a colour with alpha=1 somewhere, he will just replace the colour that is already there. If alpha=1/3, it seems reasonable to use a convex
combination of 1/3 of the colour to paint (colour of the source) and 2/3 of the colour that is already in place (the colour at the destination), so
α*src+(1-α)*dst (for colour and for alpha components), and we have to decide which colours to affect: 
```rust
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
```
There could be other data that we want to pass to the pipeline, which is not attached to vertices. If so, these go into the PipelineLayout. We will
certainly come back to it, but for now, there is nothing: 
```rust
        let pipelinelayout_info = vk::PipelineLayoutCreateInfo::builder();
        let pipelinelayout =
            unsafe { logical_device.create_pipeline_layout(&pipelinelayout_info, None) }?;
```
Aaaand ... we can create the pipeline. In addition to the long list of things just discussed, we have to indicate renderpass and subpass (that's why
we created them earlier): 
```rust
        let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&shader_stages)
            .vertex_input_state(&vertex_input_info)
            .input_assembly_state(&input_assembly_info)
            .viewport_state(&viewport_info)
            .rasterization_state(&rasterizer_info)
            .multisample_state(&multisampler_info)
            .color_blend_state(&colourblend_info)
            .layout(pipelinelayout)
            .render_pass(renderpass)
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
```
This time, the "create" command expects to create several pipelines at once (an array of `PipelineCreateInfo`s turns into an equal amount of Pipelines).
There often will be many pipelines in a program (it is not possible to change one little thing; every change means creating a new pipeline (note: not
entirely true, but close enough to reality)), and
setting one up is a comparatively expensive thing to do. Therefore: Creating a lot of pipelines at startup (or during loading screens) seems to be the
thing to do. We even could provide some pipeline cache with pipelines created earlier (even during previous runs of the application), which is another
thing we don't do: Only one pipeline for now. 

And, of course, there is again some cleanup to do: 
```rust
        unsafe {
            logical_device.destroy_shader_module(fragmentshader_module, None);
            logical_device.destroy_shader_module(vertexshader_module, None);
        }
        unsafe {
            logical_device.destroy_pipeline(graphicspipeline, None);
            logical_device.destroy_pipeline_layout(pipelinelayout, None);
        }
```
We can destroy the shader modules as soon as we are done with creating the pipeline; we keep pipeline and pipeline layout around until the end of the
program. On that note, let's put them into a separate struct.
```rust
struct Pipeline {
    pipeline: vk::Pipeline,
    layout: vk::PipelineLayout,
}

impl Pipeline {
    fn cleanup(&self, logical_device: &ash::Device) {
        unsafe {
            logical_device.destroy_pipeline(self.pipeline, None);
            logical_device.destroy_pipeline_layout(self.layout, None);
        }
    }

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
}
```
And naturally, we will add this pipeline as another field to our Aetna struct 
```rust
        let pipeline = Pipeline::init(&logical_device, &swapchain, &renderpass)?;

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
        })
```
and modify the drop function: 
```rust
impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
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


[Continue](010_Commandbuffers.md)
