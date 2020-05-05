## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Positioning the boxes, rotation and scaling

We have several boxes in our scene. We can put them in different places. But can we also have them in different sizes? And possibly rotated? 

It would be good if they could still share their vertex data. 

Therefore: This will become part of the instance data. 

That seems logical: There we are also storing the position (offset) of the whole model. 

How can we include scaling and rotation? 

Well, both are linear transformations. And together with the translation they fit into one 4x4 matrix. (To be fair, here we *could* get away with fewer entries than the
full matrix. We don't try.)

What do we do with this matrix (let's call it 
<img src="svg/022_-868c5517f5b16e98b7725121eaea53bff91bfc05643123d1c556b6fa5b76cc9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.20456pt" height="22.36361pt"/>)? We multiply every vertex position 
<img src="svg/022_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> (remember: A position is a vector with four components) of our model by this matrix when drawing. That is, we draw the vertex at 
<img src="svg/022_-8e2a8e22cb76e778eaf1cde8228b56dc9e3f8a6a586908a8f109667f9fc9782e.svg?sanitize=true?invert_in_darkmode" align="middle" width="30.723501pt" height="22.36361pt"/>, not at 
<img src="svg/022_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>.

If 
<img src="svg/022_-868c5517f5b16e98b7725121eaea53bff91bfc05643123d1c556b6fa5b76cc9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.20456pt" height="22.36361pt"/> encodes a translation by some vector 
<img src="svg/022_-ef5db0f489b15cbf3832662f168e42c98e00b502f0819c4453853d40d8f746b5.svg?sanitize=true?invert_in_darkmode" align="middle" width="10.454576pt" height="20.129883pt"/>, for example, we will draw the point that was described as being at the origin when we created the
model at 
<img src="svg/022_-ef5db0f489b15cbf3832662f168e42c98e00b502f0819c4453853d40d8f746b5.svg?sanitize=true?invert_in_darkmode" align="middle" width="10.454576pt" height="20.129883pt"/> instead. 
<img src="svg/022_-868c5517f5b16e98b7725121eaea53bff91bfc05643123d1c556b6fa5b76cc9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.20456pt" height="22.36361pt"/> translates the coordinates from a model-local system (origin = centre of the model, or maybe: the point in the middle below
its "feet") to a world coordinate system (a global coordinate system in which we describe the positions off all objects, origin = some special point
in space relative to which positions shall be given).  

This matrix is usually called "model matrix". How to figure out its components? Think about what the vectors 
<img src="svg/022_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> etc. should be transformed to, and
write that as columns of the matrix (cf. chapters [14](014_Vectors_Matrices.md) and following). 



So instead of the `position_offset` we should pass a whole 4x4 matrix into our vertex shader, and then apply this matrix to the position vector in the
shader. There is a suitable variable type in GLSL: `mat4` 

```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in mat4 model_matrix;
layout (location=2) in vec3 colour;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_Position = model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
}
```

We have learned that the next thing to adapt are the vertex attribute descriptions. For the position offset of `[f32; 3]`, it was
```rust
            vk::VertexInputAttributeDescription {
                binding: 1,
                location: 1,
                offset: 0,
                format: vk::Format::R32G32B32_SFLOAT,
            },
```
and we certainly have to change the format. But if we look at the [list of formats](https://docs.rs/ash/0.30.0/ash/vk/struct.Format.html), we have to
notice that there is not really a suitable `[f32; 16]` variant (which would be what we'd need for our 4x4 matrix). 

What do we do? Well, we can just use several locations and transmit the matrix column by column. (If it's

<img src="svg/022_-ab9a5b6fbcd1ac98e2bd4326648260e3167fc88b3f637ef0be73e8811434325d.svg?sanitize=true?invert_in_darkmode" align="middle" width="146.19562pt" height="86.72797pt"/>, we send `[1,5,9,13]` first, then `[2,6,10,14]` and so on.) 

```rust
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
                format: vk::Format::R32G32B32_SFLOAT,
            },
        ];
        let vertex_binding_descs = [
            vk::VertexInputBindingDescription {
                binding: 0,
                stride: 12,
                input_rate: vk::VertexInputRate::VERTEX,
            },
            vk::VertexInputBindingDescription {
                binding: 1,
                stride: 76,
                input_rate: vk::VertexInputRate::INSTANCE,
            },
        ];
```
(Locations 1,2,3,4 are the matrix.)

We have to correct the shader: 
```glsl
#version 450

layout (location=0) in vec3 position;
layout (location=1) in mat4 model_matrix;
layout (location=5) in vec3 colour;

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_Position = model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
}
```
Note that we are still keeping the `mat4` as location 1 here. (Actually, it's locations 1, 2, 3 and 4, but they will be automatically combined.) What
we had to adapt was the location for colour: It cannot be location 2, it has to use the next "free" location (compare with the
`VertexInputAttributeDescription`s). 

Next place to change: our `InstanceData` struct: 
```rust
#[repr(C)]
struct InstanceData {
    modelmatrix: [f32; 16],
    colour: [f32; 3],
}
```
as well as all places where it is created: 
```rust
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0,
            ],
            colour: [1.0, 0.0, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.25, 0.0, 1.0,
            ],
            colour: [0.6, 0.5, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.5, 0.0, 1.0,
            ],
            colour: [0.0, 0.5, 0.0],
        });
```
These changes are enough to have the program running again, with the same output. 

(If the output has become a black window by these changes, I recommend checking the `offset`s and `stride`s in the vertex attribute and binding
descriptions first.) 


I don't really like 
```rust
            modelmatrix: [
                1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0,
            ],
```
very much. Maybe we can make it a bit more readable by using `[[f32; 4]; 4]` instead of `[f32; 16]`: 
```rust
#[repr(C)]
struct InstanceData {
    modelmatrix: [[f32; 4]; 4],
    colour: [f32; 3],
}
```
and 
```rust
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                [1.0, 0.0, 0.0, 0.0],
                [0.0, 1.0, 0.0, 0.0],
                [0.0, 0.0, 1.0, 0.0],
                [0.0, 0.0, 0.0, 1.0],
            ],
            colour: [1.0, 0.0, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                [1.0, 0.0, 0.0, 0.0],
                [0.0, 1.0, 0.0, 0.0],
                [0.0, 0.0, 1.0, 0.0],
                [0.0, 0.25, 0.0, 1.0],
            ],
            colour: [0.6, 0.5, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: [
                [1.0, 0.0, 0.0, 0.0],
                [0.0, 1.0, 0.0, 0.0],
                [0.0, 0.0, 1.0, 0.0],
                [0.0, 0.5, 0.0, 1.0],
            ],
            colour: [0.0, 0.5, 0.0],
        });
```
These are the only changes required: How those `f32`s are stored in memory is unaffected; our vertex shader (or vertex buffer) won't notice any difference. 

However, this way is still confusing. The *lines* in this code are actually *columns* of the matrices, which should be easy to froget. Also, we
can't really use these matrices for common operations like matrix\*matrix or matrix\*vector etc.  

We should dedicated matrix and vector types in our code. We could start implementing them ourselves — but it seems more reasonable to include a
specialized crate for linear algebra; my choice is [nalgebra](www.nalgebra.org). New line for `Cargo.toml`: `nalgebra = "0.18.0"`, and new line for our code:
`use nalgebra as na;`

Creating the matrix then can take different forms: 
```rust
        let matrix1 = na::Matrix4::from_column_slice(&[
            1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.5, 0.0, 1.0,
        ]);
        let matrix2 = na::Matrix4::from_row_slice(&[
            1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.5, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0,
        ]);
        let matrix3 = na::Matrix4::<f32>::new(
            1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.5, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0,
        );
        let matrix4 = na::Matrix4::from_columns(&[
            na::Vector4::new(1.0, 0.0, 0.0, 0.0),
            na::Vector4::new(0.0, 1.0, 0.0, 0.0),
            na::Vector4::new(0.0, 0.0, 1.0, 0.0),
            na::Vector4::new(0.0, 0.5, 0.0, 1.0),
        ]);
        let matrix5 = na::Matrix4::from_rows(&[
            na::RowVector4::new(1.0, 0.0, 0.0, 0.0),
            na::RowVector4::new(0.0, 1.0, 0.0, 0.5),
            na::RowVector4::new(0.0, 0.0, 1.0, 0.0),
            na::RowVector4::new(0.0, 0.0, 0.0, 1.0),
        ]);
        let matrix6 = na::Matrix4::<f32>::new(
            1.0, 0.0, 0.0, 0.0, //
            0.0, 1.0, 0.0, 0.5, //
            0.0, 0.0, 1.0, 0.0, //
            0.0, 0.0, 0.0, 1.0,
        );
        println!("{}", matrix1);
        println!("{}", matrix2);
        println!("{}", matrix3);
        println!("{}", matrix4);
        println!("{}", matrix5);
        println!("{}", matrix6);
```
All of these matrices are the same (and the same as the last matrix before). I recommend `println!` over `dbg!` for inspecting it. 

A small "trick" you can see in this code example are the (empty) comments at the ends of the lines in the definition of `matrix6` tricking rustfmt
into leaving these lines alone and keeping them in a readable "looks just like a regular matrix" form. (It doesn't work any longer if the matrix
entries become so long that rustfmt wants to put them in separate lines anyway.)

It's worth noting that nalgebra comes with a bunch of additional types that carry information like "this vector has unit length" or "this
transformation is a rotation (and a rotation only)" and so on.

— If we want to construct the same matrix as before in a more self-explanatory way:
```rust
        let matrix7 = na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0));
```

We can also turn these matrices in the right form to use with our shaders: 
```rust
        let matrix_for_vulkan: [[f32; 4]; 4] = matrix7.into();
```
(We could also rely on `.to_slice` methods or adjust the type of `modelmatrix` in `InstanceData` to `na::Matrix4`. We do neither.) 

The creation of boxes could look like this: 
```rust
        cube.insert_visibly(InstanceData {
            modelmatrix: na::Matrix4::identity().into(),
            colour: [1.0, 0.0, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.25, 0.0)).into(),
            colour: [0.6, 0.5, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: (na::Matrix4::from_scaled_axis(na::Vector3::new(
                0.0,
                0.0,
                std::f32::consts::FRAC_PI_3,
            )) * na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0)))
            .into(),
            colour: [0.0, 0.5, 0.0],
        });
```
Two comments on the transformation for the last cube, "Rotation\*Translation": Firstly, the rotation is given as "rotation about the z-axis" (the
direction of the vector in this function), the angle (here, π/3) is encoded in the length of the given vector. Secondly, "Rotation\*Translation"
means: translate first, then rotate (consider: R\*T\*v = R\*(T\*v)). And (try it!): the order does matter (for operations like rotation and
translation, or more generally for the multiplication of matrices, except in very special cases).


In our definition of the cube
```rust
impl Model<[f32; 3], InstanceData> {
    fn cube() -> Model<[f32; 3], InstanceData> {
        let lbf = [-0.1, 0.1, 0.0]; //lbf: left-bottom-front
        let lbb = [-0.1, 0.1, 0.1];
        let ltf = [-0.1, -0.1, 0.0];
        let ltb = [-0.1, -0.1, 0.1];
        let rbf = [0.1, 0.1, 0.0];
        let rbb = [0.1, 0.1, 0.1];
        let rtf = [0.1, -0.1, 0.0];
        let rtb = [0.1, -0.1, 0.1];
        Model {
            vertexdata: vec![
                lbf, lbb, rbb, lbf, rbb, rbf, //bottom
                ltf, rtb, ltb, ltf, rtf, rtb, //top
                lbf, rtf, ltf, lbf, rbf, rtf, //front
                lbb, ltb, rtb, lbb, rtb, rbb, //back
                lbf, ltf, lbb, lbb, ltf, ltb, //left
                rbf, rbb, rtf, rbb, rtb, rtf, //right
            ],
            handle_to_index: std::collections::HashMap::new(),
            handles: Vec::new(),
            instances: Vec::new(),
            first_invisible: 0,
            next_handle: 0,
            vertexbuffer: None,
            instancebuffer: None,
        }
    }
}
```
we use coordinates `0.1`. Let us change these to `1.0` — that seems more canonical — and rather scale the models we are using. That is something we
can do with our new `InstanceData`.

```rust
        cube.insert_visibly(InstanceData {
            modelmatrix: na::Matrix4::new_scaling(0.1).into(),
            colour: [1.0, 0.0, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: (na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.25, 0.0))
                * na::Matrix4::new_scaling(0.1))
            .into(),
            colour: [0.6, 0.5, 0.0],
        });
        cube.insert_visibly(InstanceData {
            modelmatrix: (na::Matrix4::from_scaled_axis(na::Vector3::new(
                0.0,
                0.0,
                std::f32::consts::FRAC_PI_3,
            )) * na::Matrix4::new_translation(&na::Vector3::new(0.0, 0.5, 0.0))
                * na::Matrix4::new_scaling(0.1))
            .into(),
            colour: [0.0, 0.5, 0.0],
        });
```
(First scale, then translate. If we translated first, also the translation vector would be scaled.)

[Continue](023_Behind_each_other.md)


