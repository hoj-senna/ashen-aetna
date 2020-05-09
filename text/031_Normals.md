## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Normals and brightness

Our sphere did not look very spherical, more flat, 2D, not 3D. The reason? All visual clues that it should be three-dimensional were missing. 

One of the most important clues is given by the effects of light (and shadow), with different brightness of surfaces indicating whether they are
oriented toward the light source or are pointing away from it.

In order to take the orientation of a surface into account, we first have to know it.

As simplest first step, let us compute it in the vertex shader: For spheres that is simple, because pointing away from the surface is the same as
pointing away from the origin, that is, the same as the position. We normalize it (it's a direction only, length should not matter) and pass it to the
fragment shader.
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in mat4 model_matrix;
layout (location=5) in vec3 colour;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;
layout (location=1) out vec3 normal;

void main() {
    gl_Position = ubo.projection_matrix*ubo.view_matrix*model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
    normal=normalize(position);
}
```
This way of getting a normal only works for spheres  centred at the origin (the relation between position and normal is different for other shapes). 

In the fragment shader, we receive the new variable and also define some light direction (as direction to the light; for "directional lights", light
sources very far away, this can be the same direction for every point in the scene), again as a normalized vector: 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) in vec4 data_from_the_vertexshader;
layout (location=1) in vec3 normal;

void main(){
	vec3 direction_to_light=normalize(vec3(-1,-1,0));
	theColour= data_from_the_vertexshader;
}
```
Finally, we have to change the value of `theColour`, according to normal and light direction. 

It should be bright if normal and light direction coincide, and if the normal points away from the light, there should be no light. We could scale the
colour by the dot product of normal and light direction: 
```glsl
	theColour= dot(normal,direction_to_light)*data_from_the_vertexshader;
```
Or, wait, the dot product could be negative. Better: 
```glsl
	theColour= max(dot(normal,direction_to_light),0)*data_from_the_vertexshader;
```

That looks decidedly more three-dimensional than before. 

Of course, one part of the model now has disappeared completely (not so surprising if there is no light, but perhaps not desired). We can keep some
part of the colour around also there: 
```glsl
	theColour= 0.5*(1+max(dot(normal,direction_to_light),0))*data_from_the_vertexshader;
```
This is a huge cheat and we will do better later. 

For now, I think, normals should become part of the model. Let us swap `[f32; 3]` (as parameter `V` of `Model<V,I>`) for a separate type `VertexData`:
```rust
#[derive(Copy, Clone, Debug)]
#[repr(C)]
pub struct VertexData {
    pub position: [f32; 3],
    pub normal: [f32; 3],
}
```
Two helper functions that we used especially in definition of the sphere and in refining the model: 
```rust
impl VertexData {
    fn midpoint(a: &VertexData, b: &VertexData) -> VertexData {
        VertexData {
            position: [
                0.5 * (a.position[0] + b.position[0]),
                0.5 * (a.position[1] + b.position[1]),
                0.5 * (a.position[2] + b.position[2]),
            ],
            normal: normalize([
                0.5 * (a.normal[0] + b.normal[0]),
                0.5 * (a.normal[1] + b.normal[1]),
                0.5 * (a.normal[2] + b.normal[2]),
            ]),
        }
    }
}
fn normalize(v: [f32; 3]) -> [f32; 3] {
    let l = (v[0] * v[0] + v[1] * v[1] + v[2] * v[2]).sqrt();
    [v[0] / l, v[1] / l, v[2] / l]
}
```
and the adjusted creation of icosahedron and sphere (I'm completely ignoring the cube for now; it remains `->Model<[f32;3],InstanceData>`):

```rust
impl Model<VertexData, InstanceData> {
    pub fn icosahedron() -> Model<VertexData, InstanceData> {
        let phi = (1.0 + 5.0_f32.sqrt()) / 2.0;
        let darkgreen_front_top = VertexData {
            position: [phi, -1.0, 0.0],
            normal: normalize([phi, -1.0, 0.0]),
        }; //0
        let darkgreen_front_bottom = VertexData {
            position: [phi, 1.0, 0.0],
            normal: normalize([phi, 1.0, 0.0]),
        }; //1
        let darkgreen_back_top = VertexData {
            position: [-phi, -1.0, 0.0],
            normal: normalize([-phi, -1.0, 0.0]),
        }; //2
        let darkgreen_back_bottom = VertexData {
            position: [-phi, 1.0, 0.0],
            normal: normalize([-phi, 1.0, 0.0]),
        }; //3
        let lightgreen_front_right = VertexData {
            position: [1.0, 0.0, -phi],
            normal: normalize([1.0, 0.0, -phi]),
        }; //4
        let lightgreen_front_left = VertexData {
            position: [-1.0, 0.0, -phi],
            normal: normalize([-1.0, 0.0, -phi]),
        }; //5
        let lightgreen_back_right = VertexData {
            position: [1.0, 0.0, phi],
            normal: normalize([1.0, 0.0, phi]),
        }; //6
        let lightgreen_back_left = VertexData {
            position: [-1.0, 0.0, phi],
            normal: normalize([-1.0, 0.0, phi]),
        }; //7
        let purple_top_left = VertexData {
            position: [0.0, -phi, -1.0],
            normal: normalize([0.0, -phi, -1.0]),
        }; //8
        let purple_top_right = VertexData {
            position: [0.0, -phi, 1.0],
            normal: normalize([0.0, -phi, 1.0]),
        }; //9
        let purple_bottom_left = VertexData {
            position: [0.0, phi, -1.0],
            normal: normalize([0.0, phi, -1.0]),
        }; //10
        let purple_bottom_right = VertexData {
            position: [0.0, phi, 1.0],
            normal: normalize([0.0, phi, 1.0]),
        }; //11
        Model {
            vertexdata: vec![
                darkgreen_front_top,
                darkgreen_front_bottom,
                darkgreen_back_top,
                darkgreen_back_bottom,
                lightgreen_front_right,
                lightgreen_front_left,
                lightgreen_back_right,
                lightgreen_back_left,
                purple_top_left,
                purple_top_right,
                purple_bottom_left,
                purple_bottom_right,
            ],
            indexdata: vec![
                0, 9, 8, //
                0, 8, 4, //
                0, 4, 1, //
                0, 1, 6, //
                0, 6, 9, //
                8, 9, 2, //
                8, 2, 5, //
                8, 5, 4, //
                4, 5, 10, //
                4, 10, 1, //
                1, 10, 11, //
                1, 11, 6, //
                2, 3, 5, //
                2, 7, 3, //
                2, 9, 7, //
                5, 3, 10, //
                3, 11, 10, //
                3, 7, 11, //
                6, 7, 9, //
                6, 11, 7, //
            ],
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
    pub fn sphere(refinements: u32) -> Model<VertexData, InstanceData> {
        let mut model = Model::icosahedron();
        for _ in 0..refinements {
            model.refine();
        }
        for v in &mut model.vertexdata {
            v.position = normalize(v.position);
        }
        model
    }
    pub fn refine(&mut self) {
        let mut new_indices = vec![];
        let mut midpoints = std::collections::HashMap::<(u32, u32), u32>::new();
        for triangle in self.indexdata.chunks(3) {
            let a = triangle[0];
            let b = triangle[1];
            let c = triangle[2];
            let vertex_a = self.vertexdata[a as usize];
            let vertex_b = self.vertexdata[b as usize];
            let vertex_c = self.vertexdata[c as usize];
            let mab = if let Some(ab) = midpoints.get(&(a, b)) {
                *ab
            } else {
                let vertex_ab = VertexData::midpoint(&vertex_a, &vertex_b);
                let mab = self.vertexdata.len() as u32;
                self.vertexdata.push(vertex_ab);
                midpoints.insert((a, b), mab);
                midpoints.insert((b, a), mab);
                mab
            };
            let mbc = if let Some(bc) = midpoints.get(&(b, c)) {
                *bc
            } else {
                let vertex_bc = VertexData::midpoint(&vertex_b, &vertex_c);
                let mbc = self.vertexdata.len() as u32;
                midpoints.insert((b, c), mbc);
                midpoints.insert((c, b), mbc);
                self.vertexdata.push(vertex_bc);
                mbc
            };
            let mca = if let Some(ca) = midpoints.get(&(c, a)) {
                *ca
            } else {
                let vertex_ca = VertexData::midpoint(&vertex_c, &vertex_a);
                let mca = self.vertexdata.len() as u32;
                midpoints.insert((c, a), mca);
                midpoints.insert((a, c), mca);
                self.vertexdata.push(vertex_ca);
                mca
            };
            new_indices.extend_from_slice(&[mca, a, mab, mab, b, mbc, mbc, c, mca, mab, mbc, mca]);
        }
        self.indexdata = new_indices;
    }
}
```
Then we have to write `Model<VertexData,InstanceData>` also in the definition of `Aetna`, and we have to include the normal in the data sent to the
vertex shader, where this means changes to the "location" attributes: 
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in mat4 model_matrix;
layout (location=6) in vec3 colour;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;
layout (location=1) out vec3 out_normal;

void main() {
    gl_Position = ubo.projection_matrix*ubo.view_matrix*model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
    out_normal=normal;
}
```
Hand in hand with changes in the shader variables goes the change of `Pipeline::init()`:
```rust
        let vertex_attrib_descs = [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                offset: 0,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription { // <--- new, for the normal 
                binding: 0,
                location: 1,
                offset: 12,
                format: vk::Format::R32G32B32_SFLOAT,
            },
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 2, // <--- adjusted, to avoid duplication. (Same for the following) 
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
                format: vk::Format::R32G32B32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 24, // <--- Don't forget the change here.
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 76,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
```

This already works.

We should think about this line: 
```glsl
    out_normal=normal;
```
Here we are just transferring the normal passed to the vertex shader over to the fragment shader. But we did something different with the position: We
first transformed the position by some matrices (model, view and projection matrix). 

Probably, we should also transform the normal. Without modifications, it is the normal relative to the model, the normal in model space. The light
direction we define in the fragment shader is ... well, at least not relative to each model, because it is fixed there, independently of the model. It
is either in world space or in view space (the latter would be "always from the top left of the screen, no matter how we move the camera", the former
"from the top left; but if we move forward, through the sphere, and then turn around, it would be from the top right"). 

Let's say it is world space. Then we still have to apply some transformation related to the model matrix. But which matrix exactly? 

Normals are defined by an orthogonality relation, orthogonality is closely connected with scalar products. Let us, therefore, start by the following
observation how scalar products interact with matrices: 


<img src="svg/031_-ae21aaa27840b9b75e0aa00e5e47245964c3c53f401f4b5a5b8b458abe73d5df.svg?sanitize=true?invert_in_darkmode" align="middle" width="120.224785pt" height="28.215157pt"/>, because


<p align="center"><img src="svg/031_-1b35391db84e4ee71c3f1afe7f149ed9abb03de1e0fe204e68421d9448557065.svg?sanitize=true?invert_in_darkmode" align="middle" width="518.42786pt" height="58.909664pt"/></p>


Now, if we have two points, 
<img src="svg/031_-2d82e615dc35edba1552130bb4fbaaa0807b0e9e2d256f3579c58a89c000a447.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.129707pt" height="14.090904pt"/> and 
<img src="svg/031_-6ec0ad5c576d88cf95f33c5b154da55daac637297c11341b7828639a25246cf2.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.129707pt" height="14.090904pt"/> on the surface of the model and, say, they are so close to each other that the connecting line

<img src="svg/031_-6b20949f1ae1b90702a34f8082b7ce6c584694d501ac8b8df0b4e038720aea90.svg?sanitize=true?invert_in_darkmode" align="middle" width="84.512085pt" height="19.09092pt"/> also lies on the surface, then the normal 
<img src="svg/031_-d646f91aebb23242b20d077356881384db27ece57cb7a8509c103e0843685993.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.367487pt" height="14.090904pt"/> is orthogonal to 
<img src="svg/031_-29e2d5ef24652652d061c8d21f8bccf9660de63eb5c2bdffd9685ef30750ca16.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.778433pt" height="14.090904pt"/>: 
<img src="svg/031_-221de848ba159d49834be7d8111c09c4f636720f22104097d1c6cd5e13dfd4c5.svg?sanitize=true?invert_in_darkmode" align="middle" width="64.41842pt" height="21.090912pt"/>. 

If we transform the model by some matrix M, we turn 
<img src="svg/031_-2d82e615dc35edba1552130bb4fbaaa0807b0e9e2d256f3579c58a89c000a447.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.129707pt" height="14.090904pt"/> and 
<img src="svg/031_-6ec0ad5c576d88cf95f33c5b154da55daac637297c11341b7828639a25246cf2.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.129707pt" height="14.090904pt"/> into 
<img src="svg/031_-3129bd437ba180b8243aafc492f56c43376bd68d627fa12da26538f84c5aa76a.svg?sanitize=true?invert_in_darkmode" align="middle" width="36.788788pt" height="22.36361pt"/> and 
<img src="svg/031_-5ee851ae1cec3183d8cdc4152dfc7ca0e42ed7eb36d6eadf1152678ec5abe0cb.svg?sanitize=true?invert_in_darkmode" align="middle" width="36.788788pt" height="22.36361pt"/> and hence 
<img src="svg/031_-29e2d5ef24652652d061c8d21f8bccf9660de63eb5c2bdffd9685ef30750ca16.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.778433pt" height="14.090904pt"/> into 
<img src="svg/031_-7d5c17c1acdb651eef3b2a80b8077d2a7924b85ff9ec8f161cf99034cfcb9b4b.svg?sanitize=true?invert_in_darkmode" align="middle" width="138.23653pt" height="22.36361pt"/>. By which matrix (say,
A) do we have to transform n? 

We want: 
<img src="svg/031_-f8a49985401634495cf98dd577675acc02b0618ceb38fdcb1629d6d1bd9d5d6e.svg?sanitize=true?invert_in_darkmode" align="middle" width="116.041466pt" height="22.36361pt"/> (the angle before and after transformation should coincide (in this case: both be right angles)). According to our observation on matrices and scalar products: 
<img src="svg/031_-09ef91e9d403240fed2835f2a40b56c5bcfe277cd52ca3689f44106df4a0f0c5.svg?sanitize=true?invert_in_darkmode" align="middle" width="155.88004pt" height="28.215157pt"/>. Therefore:
If 
<img src="svg/031_-661116e4171adc6607c70dd1cd461efc5a15db9b6539874077c17c532de3655d.svg?sanitize=true?invert_in_darkmode" align="middle" width="25.977753pt" height="28.215157pt"/> is the inverse of 
<img src="svg/031_-868c5517f5b16e98b7725121eaea53bff91bfc05643123d1c556b6fa5b76cc9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.20456pt" height="22.36361pt"/>, then the angle between the transformed normal and points is the same as the angle before transformation. 

We should, therefore, transform the normal by 
<img src="svg/031_-db15dcb1f0b68d97fb6ed690ad2a1652fa0b1144322c6c545c40c8d351fec08c.svg?sanitize=true?invert_in_darkmode" align="middle" width="154.42407pt" height="28.215157pt"/>, the transpose of the inverse of M (or the inverse of the transpose, that's the
same). — Note that 
<img src="svg/031_-3f388243a81d8c3f9c3bab4f35fe5d752f9d74c534bd7a5062399fb6177fb743.svg?sanitize=true?invert_in_darkmode" align="middle" width="41.24384pt" height="28.215157pt"/> is just the introduction of new notation, "-T" is meaningless otherwise. 

Let me repeat: The normal is to be transformed by the transposed inverse, not by the same matrix as the points. 

(This distinction is moot for pure rotations. We have already seen that for those inversion and transpose is the same; so 
<img src="svg/031_-c5eb40f2c2810d7f9f6813bb3a65a0f6523fd510677d723cf9a0c565612375a4.svg?sanitize=true?invert_in_darkmode" align="middle" width="81.468185pt" height="28.215157pt"/>. But that's just
a special case.) 

For our shader, that means: 
```glsl
    out_normal=vec3(transpose(inverse(model_matrix))*vec4(normal,0.0));
```
In general, we don't want to invert matrices. Especially not in shaders which are supposed to be called very often. It would be better to make the
inverted model matrix part of the InstanceData, for example, and submit it at the same time as the model matrix itself.

Let's do that: 
```rust
#[repr(C)]
pub struct InstanceData {
    pub modelmatrix: [[f32; 4]; 4],
    pub inverse_modelmatrix: [[f32; 4]; 4],
    pub colour: [f32; 3],
}
impl InstanceData {
    pub fn from_matrix_and_colour(modelmatrix: na::Matrix4<f32>, colour: [f32; 3]) -> InstanceData {
        InstanceData {
            modelmatrix: modelmatrix.into(),
            inverse_modelmatrix: modelmatrix.try_inverse().unwrap().into(),
            colour,
        }
    }
}
```
and 
```rust
    sphere.insert_visibly(InstanceData::from_matrix_and_colour(
        na::Matrix4::new_scaling(0.5),
        [0.5, 0.0, 0.0],
    ));
```
together with 
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in mat4 model_matrix;
layout (location=6) in mat4 inverse_model_matrix;
layout (location=10) in vec3 colour;

layout (set=0, binding=0) uniform UniformBufferObject {
	mat4 view_matrix;
	mat4 projection_matrix;
} ubo;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;
layout (location=1) out vec3 out_normal;

void main() {
    gl_Position = ubo.projection_matrix*ubo.view_matrix*model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
    out_normal=vec3(inverse_model_matrix*vec4(normal,0.0));
}
```
and 
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
                format: vk::Format::R32G32B32_SFLOAT,
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
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 10,
                offset: 128,
                format: vk::Format::R32G32B32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 24,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 140,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
```
(You'll have recognized where to put these.) 

Okay. Recap: The sphere looks three-dimensional; we know how to transform normals, and we have a very simple "shading model" that affects the colour
of pixels depending on the normal of their surface and the light direction. 

This model is much too basic, but much, much better than what we had before. And that's enough for now. 

(We'll have to come back and improve the shading, and do something about the fact that the light direction is just one hard-coded value in the
fragment shader.) 

[Continue]  ( next chapter not yet ready ) 


