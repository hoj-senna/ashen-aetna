## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Textures II: 2 Textures
 
The code from last chapter is missing one final piece of cleanup: 
```
ashen-aetna: vendor/src/vk_mem_alloc.h:11791: void VmaDeviceMemoryBlock::Destroy(VmaAllocator): Assertion `m_pMetadata->IsEmpty() && "Some allocations were not freed before destruction of this memory block!"' failed.
```
That is the texture's allocation which we did not free. 

We need some `aetna.allocator.destroy_image(texture.vk_image,&texture.allocation);`

But where should we put it? 

We cannot implement `Drop` for `Texture`, because we need some external information (the `vk_mem::Allocator`) for this destruction.

There is no "end of the main() function" where we could write this line (because winit hijacks the thread and never returns control out of the closure
in `eventloop.run()`, that is, everything written after this `run` gets ignored). We could react to the `Event::LoopDestroyed` event.

Or we could make the texture part of `Aetna` and destroy the allocation in `Aetna`'s drop function. 

I'd also like to be able to have more than one texture (see title), so — maybe — it is reasonable to invent a structure that keeps several textures, say a `TextureStorage`, 
and gets a `cleanup` function (destroying the images of all textures). We could then make this structure part of `Aetna` and call the cleanup when
dropping the volcano. 

Something along these lines: 
```rust
pub struct TextureStorage {
    textures: Vec<Texture>,
}

impl TextureStorage {
   pub fn new() -> Self {
        TextureStorage { textures: vec![] }
    }
   pub fn cleanup(&mut self, allocator: &vk_mem::Allocator) {
        for texture in &self.textures {
            allocator.destroy_image(texture.vk_image, &texture.allocation);
        }
    }
    pub fn new_texture_from_file<P: AsRef<std::path::Path>>(
        &mut self,
        path: P,
        aetna: &Aetna,
    ) -> Result<usize, Box<dyn std::error::Error>> {
        let new_texture = Texture::from_file(path, aetna)?;
        let new_id = self.textures.len();
        self.textures.push(new_texture);
        Ok(new_id)
    }
    pub fn get(&self, index: usize) -> Option<&Texture> {
        self.textures.get(index)
    }
    pub fn get_mut(&mut self, index: usize) -> Option<&mut Texture> {
        self.textures.get_mut(index)
    }
}
```

If we want to make such a texture storage part of `Aetna`, however, it is rather inconvenient to have `aetna: &Aetna` as argument of the
`new_from_file` or, in turn, of the `Texture::from_file` function.

We better split it: 
```rust
impl Texture {
    pub fn from_file<P: AsRef<std::path::Path>>(
        path: P,
        device: &ash::Device,
        allocator: &vk_mem::Allocator,
        commandpool_graphics: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> Result<Self, Box<dyn std::error::Error>> {
```
(with the obvious modifications throughout the function body), and similarly
```rust
    pub fn new_texture_from_file<P: AsRef<std::path::Path>>(
        &mut self,
        path: P,
        device: &ash::Device,
        allocator: &vk_mem::Allocator,
        commandpool_graphics: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> Result<usize, Box<dyn std::error::Error>> {
        let new_texture = Texture::from_file(
            path,
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
In `aetna.rs` we add `use crate::texture::TextureStorage;`, `Aetna` obtains a new field `pub texture_storage: TextureStorage,` and the return value of
`Aetna::init()` gains the line `texture_storage: TextureStorage::new(),`.

In `Aetna::drop()`, as intenden, we call the `TextureStorage::cleanup`:
```rust
impl Drop for Aetna {
    fn drop(&mut self) {
        unsafe {
            self.device
                .device_wait_idle()
                .expect("something wrong while waiting");
            self.texture_storage.cleanup(&self.allocator);
            self.device
                .destroy_descriptor_pool(self.descriptor_pool, None);
``` 
`Aetna` should grow a function to load images.
```rust
    pub fn new_texture_from_file<P: AsRef<std::path::Path>>(
        &mut self,
        path: P,
    ) -> Result<usize, Box<dyn std::error::Error>> {
        self.texture_storage.new_texture_from_file(
            path,
            &self.device,
            &self.allocator,
            &self.pools.commandpool_graphics,
            &self.queues.graphics_queue,
        )
    }
```

Loading the texture (in `main`) then happens via 
```rust
    let texture_id = aetna.new_texture_from_file("../gfx/image.png")?;
```
And the places where we used the texture before become 
```rust
            if let Some(texture) = aetna.texture_storage.get(texture_id) {
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
            }
```
The error message is gone. It is replaced by a complaint that some `ImageView` has not been destroyed.

Updating the `TextureStorage::cleanup` function: 
```rust
    pub fn cleanup(&mut self, device: &ash::Device, allocator: &vk_mem::Allocator) {
        for texture in &self.textures {
            unsafe { device.destroy_image_view(texture.imageview, None) };
            allocator.destroy_image(texture.vk_image, &texture.allocation);
        }
    }
```
with a new `device` argument; the error disappears — and there is a new complaint by the validation layers: also the sampler has to be destroyed.
Well: 
```rust 
    pub fn cleanup(&mut self, device: &ash::Device, allocator: &vk_mem::Allocator) {
        for texture in &self.textures {
            unsafe {
                device.destroy_sampler(texture.sampler, None);
                device.destroy_image_view(texture.imageview, None);
            }
            allocator.destroy_image(texture.vk_image, &texture.allocation);
        }
    }
```

Okay. That's all problems taken care of.

Let us try to have two textures.
```rust
    let second_texture_id = aetna.new_texture_from_file("../gfx/image2.png")?;
```
If we make this variable (and the old `texture_id`) mutable, we can, for example by including these lines: 
```rust
                    winit::event::VirtualKeyCode::F11 => {
                        std::mem::swap(&mut texture_id, &mut second_texture_id);
                    }
```
in our event handling code, switch between the two textures.

Let us insert a second quad: 
```rust
    quad.insert_visibly(TexturedInstanceData::from_matrix(
        na::Matrix4::new_translation(&na::Vector3::new(2.0, 0., 0.3)),
    ));
```
Both have the same texture.

Question: Can we give to one the first, and to the other the second texture?

Answer: Yes. It has to be possible somehow. 

New question: How? 

A possible first idea: First combine the textures into one, adjust the texture coordinates in the vertices, and thus sidestep the problem. This would
work. The keyword to search for is "texture atlas".

Second idea: Have several descriptor sets, each texture bound to one of them, and intersperse some `cmd_bind_descriptor_set` commands with the
`cmd_bind_vertex_buffer` commands in `Model::draw()`. This, too, would be possible. If there is one texture that all quads have in common, and another
texture that is identical for all spheres etc., this should even be convenient. 

If I want different textures for the quad model (and want to keep it
as one `Model`), I'd, however, prefer it if I could make "which texture to use" part of the per-instance data. Therefore: 

Third idea: Bind a whole array of samplers/images and pass the index as part of the instance data. That's probably the most complicated of these ways,
but it's nevertheless the one I want to attempt.

The fragment shader receives a small change so as to allow for two textures. Let's see if it works as an array:
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec2 uv;

layout(set=1,binding=0) uniform sampler2D texturesamplers[2];

void main(){
	theColour=texture(texturesamplers[0],uv);
}
```
Well: 
```
[Debug][error][validation] "Shader expects at least 2 descriptors for binding 1.0 but only 1 provided"
```
That was to be expected. Then let's try to provide a second descriptor. During pipeline setup:
```rust
        let descriptorset_layout_binding_descs1 = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(2)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
```
(The descriptor count has been changed.) 

Now the descriptor pool is too small: 
```
[Debug][error][validation] "Unable to allocate 6 descriptors of type VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER from VkDescriptorPool 0x260000000026[]. This pool only has 3 descrip
tors of this type remaining. The Vulkan spec states: descriptorPool must have enough free descriptor capacity remaining to allocate the descriptor sets of the specified layouts (h
ttps://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#VUID-VkDescriptorSetAllocateInfo-descriptorPool-00307)"
```
Well, 
```rust
            vk::DescriptorPoolSize {
                ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
                descriptor_count: 2*swapchain.amount_of_images,
            },
```
(in `Aetna::init()`).

Time to make the texture id part of the instance data:
```rust
#[repr(C)]
pub struct TexturedInstanceData {
    pub modelmatrix: [[f32; 4]; 4],
    pub inverse_modelmatrix: [[f32; 4]; 4],
    pub texture_id: u32,
}

impl TexturedInstanceData {
    pub fn from_matrix(modelmatrix: na::Matrix4<f32>) -> TexturedInstanceData {
        TexturedInstanceData {
            modelmatrix: modelmatrix.into(),
            inverse_modelmatrix: modelmatrix.try_inverse().unwrap().into(),
            texture_id: 0,
        }
    }
    pub fn from_matrix_and_texture(
        modelmatrix: na::Matrix4<f32>,
        texture_id: usize,
    ) -> TexturedInstanceData {
        TexturedInstanceData {
            modelmatrix: modelmatrix.into(),
            inverse_modelmatrix: modelmatrix.try_inverse().unwrap().into(),
            texture_id: texture_id as u32,
        }
    }
}
```
Running the program right now, by the way, makes the second quad disappear. Why? Because we have not adjusted the information on instance data that we
tell the pipeline on its creation.
```rust
            vk::VertexInputAttributeDescription{
                binding: 1,
                location: 10,
                offset: 128,
                format: vk::Format::R8G8B8A8_UINT,
            },
```
and 
```rust
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 132,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
```
Better. But: 
```
[Debug][warning][performance] "Vertex attribute at location 10 not consumed by vertex shader"
```
That is not too surprising. Let's do that. Vertex shader: 
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in vec2 texcoord;
layout (location=2) in mat4 model_matrix;
layout (location=6) in mat4 inverse_model_matrix;
layout (location=10) in uint texture_id;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec2 uv;
layout (location=1) out uint tex_id;

void main() {
    vec4 worldpos = model_matrix*vec4(position,1.0);
    gl_Position = ubo.projection_matrix*ubo.view_matrix*worldpos;
    uv = texcoord;
    tex_id=texture_id;
}
```
And fragment shader: 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec2 uv;
layout (location=1) in uint texture_id;

layout(set=1,binding=0) uniform sampler2D texturesamplers[2];

void main(){
	theColour=texture(texturesamplers[0],uv);
}
```
We obtain a compilation error:
```
/shaders/shader_textured.frag:6: error: 'int' : must be qualified as flat in
```
What's that? 

Well, whenever one variable is not the same at all vertices of a triangle, the fragment shader receives an interpolated value, based on the position
of the fragment. We have made use of this for colours and heavily rely on it for the texture coordinates. The interpolation for integers is
problematic: How should we interpolate `1` and `2` — and still end up with an `int`? Simple solution: We don't. We use the value from the first vertex
and ignore those of the second and third. The way to tell that to the shader is to include the `flat` keyword. 
```glsl
layout (location=1) flat in uint texture_id;
```
And no, it doesn't matter that the values here will never differ between different vertices because they are part of the per-instance data. That's
something the shader does not know.

Okay, with that problem taken care of, let's use the id.
```glsl
	theColour=texture(texturesamplers[texture_id],uv);
```
And: 
```rust
    quad.insert_visibly(TexturedInstanceData::from_matrix_and_texture(
        na::Matrix4::identity(),
        texture_id,
    ));
    quad.insert_visibly(TexturedInstanceData::from_matrix_and_texture(
        na::Matrix4::new_translation(&na::Vector3::new(2.0, 0., 0.3)),
        second_texture_id,
    ));
```
... and with that we have two quads with different textures.

What about a third texture? 

Can we make it unnecessary for the shader to know how many textures we will send? 

Like this? 
```glsl
layout(set=1,binding=0) uniform sampler2D texturesamplers[];
```
Unfortunately, we then cannot use a variable index. (A similar issue arose in the chapter about lights, but we avoided it in a different way there.)
See: 
```
shader_textured.frag:12: error: 'variable index' : required extension not requested: GL_EXT_nonuniform_qualifier
```
The extension is a GLSL extension and we request it by including the following line in the shader: 
```glsl
#extension GL_EXT_nonuniform_qualifier : require
```
New error (or rather: hint by the validation layers): 
```
[Debug][error][validation] "Shader requires VkPhysicalDeviceDescriptorIndexingFeatures::runtimeDescriptorArray but is not enabled on the device"
```

Thus, on device creation, we should enable this feature. In `init_device_and_queues()` (in `intance_device_queues.rs`): 
```rust
    let indexing_features =
        vk::PhysicalDeviceDescriptorIndexingFeatures::builder().runtime_descriptor_array(true);
```
Now where to put it? The struct implements `ExtendsDeviceCreateInfo`, which means it is something we can plug in as extension to `DeviceCreateInfo`;
plugging in an extension means handing it to the corresponding `push_next`:
```rust
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_features(&features)
        .push_next(&indexing_features);
```
The next error message (this time the Rust compiler) tells us we have to use a mutable reference instead. Well, two `mut`s, and we're done.

With that, we don't have to tell the shader the number of textures. We still have to know it when creating the pipeline. Can we get rid of the number
there? 

Let us try to include some `DescriptorBindingFlags` stating that the descriptor count may be variable.
```rust
        let descriptorset_layout_binding_descs1 = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(2)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
        let descriptor_binding_flags = [vk::DescriptorBindingFlags::VARIABLE_DESCRIPTOR_COUNT];
        let mut descriptorset_layout_binding_flags =
            vk::DescriptorSetLayoutBindingFlagsCreateInfo::builder()
                .binding_flags(&descriptor_binding_flags);
        let descriptorset_layout_info1 = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs1)
            .push_next(&mut descriptorset_layout_binding_flags);
        let descriptorsetlayout1 = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info1, None)
        }?;
```
Again, this is an extension that ends up in `push_next`. It also requires activation of another device feature: 
```rust
    let mut indexing_features = vk::PhysicalDeviceDescriptorIndexingFeatures::builder()
        .runtime_descriptor_array(true)
        .descriptor_binding_variable_descriptor_count(true);
```
If we "forget" binding one of the descriptor sets:
```rust
            if let Some(texture) = aetna.texture_storage.get(texture_id) {
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
            }
            /*if let Some(texture) = aetna.texture_storage.get(second_texture_id) {
                let imageinfo = vk::DescriptorImageInfo {
                    image_layout: vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
                    image_view: texture.imageview,
                    sampler: texture.sampler,
                    ..Default::default()
                };
                let descriptorwrite_image = vk::WriteDescriptorSet {
                    dst_set: aetna.descriptor_sets_texture[aetna.swapchain.current_image],
                    dst_binding: 0,
                    dst_array_element: 1,
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
            }*/
```
there is quite a difference between the version with and without 
```rust
            .push_next(&mut descriptorset_layout_binding_flags)
```
(Okay, "quite a difference" - it's one (repeated) message from the validation layers:
```
[Debug][error][validation] "VkDescriptorSet 0x2c000000002c[] bound as set #1 encountered the following validation error at vkCmdDrawIndexed() time: Descriptor in binding #0 index 
1 is being used in draw but has never been updated via vkUpdateDescriptorSets() or a similar call."
```
— but still, better to make it disappear.)

Now let us also bind the descriptor sets jointly, and not have a long construction for every single texture separately.

We replace the long part (starting from `if let Some(texture) = aetna.texture_storage.get(texture_id) {`) by 
```rust
            let imageinfos = aetna.texture_storage.get_descriptor_image_info();
            let descriptorwrite_image = vk::WriteDescriptorSet::builder()
                .dst_set(aetna.descriptor_sets_texture[aetna.swapchain.current_image])
                .dst_binding(0)
                .dst_array_element(0)
                .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
                .image_info(&imageinfos)
                .build();
            unsafe {
                aetna
                    .device
                    .update_descriptor_sets(&[descriptorwrite_image], &[]);
            }
```
Changes: We're now using the `builder` for the creation of a descriptor set, we're updating all sets at once (`imageinfos` is a `Vec` containing
`DecriptorImageInfo`s for every texture), and we have put the collection of the descriptor image infos into a separate function (part of
`TextureStorage`):
```rust
    pub fn get_descriptor_image_info(&self) -> Vec<vk::DescriptorImageInfo> {
        self.textures
            .iter()
            .map(|t| vk::DescriptorImageInfo {
                image_layout: vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
                image_view: t.imageview,
                sampler: t.sampler,
                ..Default::default()
            })
            .collect()
    }
```
If we actually want to include a third texture, we won't have to change this place. But: The inclusion of 
```rust
    let mut third_texture_id = aetna.new_texture_from_file("../gfx/image2.png")?;
```
still provokes
```
[Debug][error][validation] "vkUpdateDescriptorSets() failed write update validation for VkDescriptorSet 0x2a000000002a[] with error: Attempting write update to descriptor set 0x2a
000000002a binding #0 with #1 descriptors being updated but this update oversteps the bounds of this binding and the next binding is not consistent with current binding so this up
date is invalid.. The Vulkan spec states: The sum of dstArrayElement and descriptorCount must be less than or equal to the number of array elements in the descriptor set binding s
pecified by dstBinding, and all applicable consecutive bindings, as described by consecutive binding updates (https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vks
pec.html#VUID-VkWriteDescriptorSet-dstArrayElement-00321)"
```
While in 
```rust
        let descriptorset_layout_binding_descs1 = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
            .descriptor_count(2)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT)
            .build()];
        let descriptor_binding_flags = [vk::DescriptorBindingFlags::VARIABLE_DESCRIPTOR_COUNT];
        let mut descriptorset_layout_binding_flags =
            vk::DescriptorSetLayoutBindingFlagsCreateInfo::builder()
                .binding_flags(&descriptor_binding_flags);
        let descriptorset_layout_info1 = vk::DescriptorSetLayoutCreateInfo::builder()
            .bindings(&descriptorset_layout_binding_descs1)
            .push_next(&mut descriptorset_layout_binding_flags);
        let descriptorsetlayout1 = unsafe {
            logical_device.create_descriptor_set_layout(&descriptorset_layout_info1, None)
        }?;
```
we said that the number may be variable, there is still a maximal number of descriptors, which we have to obey.

Okay: 
```rust
            .descriptor_count(MAXIMAL_NUMBER_OF_TEXTURES)
```
and 
```rust
const MAXIMAL_NUMBER_OF_TEXTURES: u32 = 1024;
```
(Or whatever. It would probably be reasonable to recreate the pipeline when this number is surpassed. (Or even: to create it only after the number of
textures is known.) But for this simple playing around this should be good enough.) 

Of course, there was another place which depended on the number: The size of the descriptor pool.

```rust
            vk::DescriptorPoolSize {
                ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
                descriptor_count: renderpass_and_pipeline::MAXIMAL_NUMBER_OF_TEXTURES
                    * swapchain.amount_of_images,
            },
```
(Note: We have to make the constant `pub` for this to work.) 

And there we are. We can deal with a large amount of textures now.

By the way, if we want the largest possible number, 
```rust
const MAXIMAL_NUMBER_OF_TEXTURES: u32 = u32::MAX;
```
will not work (at least not on my graphics card). 

But the validation layers will tell me how much is okay:
```
[Debug][error][validation] "vkCreatePipelineLayout(): max per-stage sampler bindings count (-1) exceeds device maxPerStageDescriptorSamplers limit (1048576). The Vulkan spec state
s: The total number of descriptors of the type VK_DESCRIPTOR_TYPE_SAMPLER and VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER accessible to any shader stage across all elements of pSetL
ayouts must be less than or equal to VkPhysicalDeviceLimits::maxPerStageDescriptorSamplers (https://www.khronos.org/registry/vulkan/specs/1.1-khr-extensions/html/vkspec.html#VUID-
VkPipelineLayoutCreateInfo-pSetLayouts-00287)"
```
Well, then:
```rust
pub const MAXIMAL_NUMBER_OF_TEXTURES: u32 = 1048576;

```

[Continue](039_Textures3.md)


















