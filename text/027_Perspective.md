## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Camera: Perspective

We have a camera that we can move around, but the "real 3D feeling" is not there yet. To improve this, our next goal is to introduce a perspective
projection, which transforms all objects similarly to what our eyes would do (making distant objects appear smaller etc.). This projection will, of
course, be represented by a 4x4-matrix. 

This chapter consists of two parts: First a mathematical derivation of the perspective matrix, then rather small adjustments to our program to include
it. 

How does projection work? 

The simplest model: Put a screen somewhere and draw every point on the screen where you would see it. "Seeing it" means: Where the direct line between
the eye (supposed to be at the origin, for simplicity) and said point intersects the screen:

 

<p align="center"><img src="svg/027_-b4ad05f9289631653dc0172f7d31fe56703567f10bc68638611f473bee74c070.svg?sanitize=true?invert_in_darkmode" align="middle" width="257.0747pt" height="137.46538pt"/></p>


We do not want to draw all points, just those in a certain range. (Or can you see things almost directly above yourself at the same time as things
near your feet?) 

 

<p align="center"><img src="svg/027_-77c3f5436009cf4138e17bb99a59f39121bf80dd0157ab3e7d14880b7aeb4c52.svg?sanitize=true?invert_in_darkmode" align="middle" width="188.60858pt" height="175.79134pt"/></p>


One way to describe that range is to indicate the opening angle φ (or "field of view in y-direction", "fovy"). 

This also gives us a hint where we should place this screen: We know that the largest y-value Vulkan will show us is 1. We should choose the distance
d between eye and screen such that the point on the line with angle φ/2 to the z-axis ends up at y=1: 

 

<p align="center"><img src="svg/027_-5fa59dc8030c7b4868ef602f89d685198bfa9532fa4be4c8521ad4571f5abfd3.svg?sanitize=true?invert_in_darkmode" align="middle" width="188.60858pt" height="175.19342pt"/></p>


Apparently, 
<img src="svg/027_-b9c98c31f6384af4aa588bc51c9afce7fe17206168cb6bda513774a11a7e8fbb.svg?sanitize=true?invert_in_darkmode" align="middle" width="86.574936pt" height="28.294647pt"/>. 

But back to the point(s): Where would a point of coordinates (y,z) end up? Well, scaling all components by the same factor keeps points on the same
line through the origin, and we want 
<img src="svg/027_-a0938bddfc9cede1960cac1c35c5a8ec8866e20fd533e41625c1ddefe0fe3950.svg?sanitize=true?invert_in_darkmode" align="middle" width="43.210167pt" height="22.72728pt"/>, so 
<img src="svg/027_-262c9e91965e520c54cac374f33d0f2e3d135a1c1cbf0ab82ee8894e5a6d18bc.svg?sanitize=true?invert_in_darkmode" align="middle" width="51.87372pt" height="25.08934pt"/>, I guess. Or, in 3D (same idea, including it in the pictures does not help): a point at
(x,y,z) lands in 
<img src="svg/027_-2b74a2eeed29f2960fa18b5caebe3b8b3fc0f2acb4c23a087cd7dc0a882a2cdc.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.40037pt" height="25.08934pt"/>. 

Now, we had agreed to use homogeneous coordinates for points in 3D graphics (look back to [Chapters 15](015_Not_so_linear.md) [and
16](016_Homogeneous_Coordinates.md)): In those terms it is 


<p align="center"><img src="svg/027_-05f5869fd464bc5c655792d998eb9dd1d1aec0ae03e91e49a3e22ff1f21eceaa.svg?sanitize=true?invert_in_darkmode" align="middle" width="47.890373pt" height="78.54619pt"/></p>

and this is [the same point](016_Homogeneous_Coordinates.md) as 


<p align="center"><img src="svg/027_-6801d53cf34ea05625fea40887c5edde928a80b4b76c08c0d38924c60bc34b6b.svg?sanitize=true?invert_in_darkmode" align="middle" width="46.505753pt" height="78.54619pt"/></p>

 
(which is a much better representation, because now we don't have to divide, and actually can write the transformation (of the original point

<img src="svg/027_-23bdc492e27539705c5b0a1fd8616a05e0a1757421edab1e14e6983a1d0dad7f.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261383pt" height="86.72797pt"/>) as matrix, via


<p align="center"><img src="svg/027_-6ec37a3edcaf1e07cc9c2634908931d7482573684ef654b1f038b8ef82e453ce.svg?sanitize=true?invert_in_darkmode" align="middle" width="298.82605pt" height="78.54619pt"/></p>


This takes all points, no matter how far away, and puts them onto the screen at z=d. Now, that is correct for the final projection onto a screen, but
we are interested in transporting points to the screen "box" [-1,1]x[-1,1]x[0,1], not [-1,1]x[-1,1]x{d}. Also, we don't need all points, no matter how
far away. In fact, let us restrict the view area by a "near plane" at z=n and a "far plane" at z=f:

  

<p align="center"><img src="svg/027_-2faca64ac43b46591f314df0a23fec5fb7415b1e8cfdd00ee94b897c2d5525ba.svg?sanitize=true?invert_in_darkmode" align="middle" width="188.60858pt" height="165.1614pt"/></p>


How do we have to change the projection? Something with the z component: Instead of dz it should be, maybe, some mz+b (for coefficients m and b that
we still have to find), that is: 


<p align="center"><img src="svg/027_-6d5555a1a43cf58396dd6db375c446bbc7cadb579c53f2955843824b04ea50e7.svg?sanitize=true?invert_in_darkmode" align="middle" width="147.6482pt" height="78.54619pt"/></p>

What are the right values for m and b? Well, z=n should be sent to 0, z=f should end up at 1, so:


<p align="center"><img src="svg/027_-2dd69261e034627507fe550843c6ce2a9f33cb9b6ec9085d565f4aa1b0d44d18.svg?sanitize=true?invert_in_darkmode" align="middle" width="186.01123pt" height="14.545458pt"/></p>

The first equation shows that 
<img src="svg/027_-167f2b33fc3610becc18e9bbc80cd86cd2fa804785817c9ca9925218131a5030.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.9589pt" height="29.490143pt"/>, and substracting the first from the second gives 
<img src="svg/027_-0e7a961f03e37eeb8f03e9241612f2f0d8fffdd85a4c5f72df6d3e1108e42fae.svg?sanitize=true?invert_in_darkmode" align="middle" width="96.59585pt" height="28.294647pt"/>, so 
<img src="svg/027_-8f683efca06c65b3de9dc1331f54749c22e5db90bb8ece08a2a9e2e8c0e846f9.svg?sanitize=true?invert_in_darkmode" align="middle" width="72.91405pt" height="31.399017pt"/> and 
<img src="svg/027_-0629d1853c2347c77c84c811f2996f928fad4280324bfd161af0208bb6afbd80.svg?sanitize=true?invert_in_darkmode" align="middle" width="67.531494pt" height="31.399017pt"/>,
which makes our matrix 



<p align="center"><img src="svg/027_-00f60f0b3258d3c0e71264c3e96dfb9244d2ae0bd94a4416dd32cb30af1238f4.svg?sanitize=true?invert_in_darkmode" align="middle" width="308.09695pt" height="78.72929pt"/></p>

We're done, if our screen is a square. Most screens aren't (and I think the window we have created is of size 800x600 pixels), so we should do
something about that. Doing something about that means scaling the x-component. We take the "aspect ratio" a (width divided by height) and scale the
x-component: 


<p align="center"><img src="svg/027_-5fa081ed34ba9fcec35b847f8eb76c9f85574d3d66d6c6b0511a3bb0aacbc885.svg?sanitize=true?invert_in_darkmode" align="middle" width="309.91345pt" height="81.85148pt"/></p>

(Now x values can be a-times larger and still fit on the screen, i.e. end up in [-1,1]x[-1,1]x[0,1].) 


  

<p align="center"><img src="svg/027_-530ece5422a4937636cef3e22e0fb5b2ab27e61fcbb4175188e3d04e29ca678f.svg?sanitize=true?invert_in_darkmode" align="middle" width="570.7125pt" height="150.01787pt"/></p>

(Sorry, you have to imagine the x-coordinates (too time-consuming); they would make the shape in the left the frustum of a pyramid; and it's actually called "view
frustum".)

The parameters we needed were: field-of-view angle, aspect ratio, and distance of the near and far plane.

```rust
impl Camera {
    fn update_projectionmatrix(&mut self) {
        let d = 1.0 / (0.5 * self.fovy).tan();
        self.projectionmatrix = na::Matrix4::new(
            d / self.aspect,
            0.0,
            0.0,
            0.0,
            0.0,
            d,
            0.0,
            0.0,
            0.0,
            0.0,
            self.far / (self.far - self.near),
            -self.near * self.far / (self.far - self.near),
            0.0,
            0.0,
            1.0,
            0.0,
        );
    }
```
and 
```rust
struct Camera {
    viewmatrix: na::Matrix4<f32>,
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
    projectionmatrix: na::Matrix4<f32>,
}
impl Default for Camera {
    fn default() -> Self {
        let mut cam = Camera {
            viewmatrix: na::Matrix4::identity(),
            position: na::Vector3::new(0.0, 0.0, 0.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 0.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 0.0)),
            fovy: std::f32::consts::FRAC_PI_3,
            aspect: 800.0 / 600.0,
            near: 0.1,
            far: 100.0,
            projectionmatrix: na::Matrix4::identity(), 
        };
        cam.update_projectionmatrix();
        cam.update_viewmatrix();
        cam
    }
}
```
(I have made sure to call the `update_matrix` functions, so that I can just set `identity` matrices.) 

Okay. (We could also add some functions to set `fovy` and automatically update `projectionmatrix`. But it's more important to use this matrix for changing our view.)

If we want to have a quick look, we can squeeze this matrix just into the view matrix. If we replace the last line of `update_viewmatrix` by 
```rust
        self.viewmatrix = self.projectionmatrix * m;
```
we can have a first look at what we have achieved. This would be a viable strategy to get the perspective information to the shader, but I want to
keep both matrices separate. We return this line to 
```rust
        self.viewmatrix =  m;
```
and get to work elsewhere. In the shader, we include another variable of the `UniformBufferObject`, and we also use it during the computation:
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

void main() {
    gl_Position = ubo.projection_matrix*ubo.view_matrix*model_matrix*vec4(position,1.0);
    colourdata_for_the_fragmentshader=vec4(colour,1.0);
}
```
If we add the matrix this way, as another member of the struct we already had, we do not have to change the descriptor set layout:
```rust
        let descriptorset_layout_binding_descs = [vk::DescriptorSetLayoutBinding::builder()
            .binding(0)
            .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
            .descriptor_count(1)
            .stage_flags(vk::ShaderStageFlags::VERTEX)
            .build()];
```
Everything still seems to fit.

The uniform buffer should be twice as large: 
```rust
        let mut uniformbuffer = Buffer::new(
            &allocator,
            128,
            vk::BufferUsageFlags::UNIFORM_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
```
and also the range for the descriptor set bindings doubles to 128 bytes size: 
```rust
            let buffer_infos = [vk::DescriptorBufferInfo {
                buffer: uniformbuffer.buffer,
                offset: 0,
                range: 128,
            }];
```
And finally, we have to pay attention to the places where we fill the uniform buffer: 
For example, we can turn this
```rust
        let cameratransform: [[f32; 4]; 4] = [
            na::Matrix4::identity().into(),
        ];
        uniformbuffer.fill(&allocator, &cameratransform)?;
```
into 
```rust
        let cameratransforms: [[[f32; 4]; 4]; 2] = [
            na::Matrix4::identity().into(),
            na::Matrix4::identity().into(),
        ];
        uniformbuffer.fill(&allocator, &cameratransforms)?;
```
Also in the `Camera::update_buffer` function: 
```rust
    fn update_buffer(&self, allocator: &vk_mem::Allocator, buffer: &mut Buffer) {
        let data: [[[f32; 4]; 4]; 2] = [self.viewmatrix.into(), self.projectionmatrix.into()];
        buffer.fill(allocator, &data);
    }
```
There are other ways to combine the two matrices and send the slice with their values to `buffer.fill()` — but I'm content with this version. 

With this camera setup, we're always in the middle of things. That's natural: It is very easy to set up models at (0,0,0), which is where our camera
is located. Maybe we should use another default position: 

```rust
            position: na::Vector3::new(-3.0, -3.0, -3.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(1.0, 1.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(-1.0, 2.0, -1.0)),
```
or 
```rust
            position: na::Vector3::new(0.0, -3.0, -3.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, -1.0)),
```
Both look at the origin, but are located outside. Going forward, I will use the latter setup. 

Small sanity check: position vector and view direction point in opposite directions, that means: looking at the origin. In both cases, view direction and down
direction are orthogonal. They also have length one. (Not the `Vector3::new` directly, but there is a `new_normalize` involved.) 


Now that I'm using it, I don't like the `Default` implementation for `Camera` any more. Default should work well in constructions like `let
cam=Camera{fovy: 1.65, ..  Default::default()};` — our `default()` does not: We *need* to know all values first and then call the `update` functions.
We cannot first construct a `default` and then change some of its values. Let us remove the `Default` implementation. 

Perhaps a `CameraBuilder` instead? 
```rust
struct CameraBuilder {
    position: na::Vector3<f32>,
    view_direction: na::Unit<na::Vector3<f32>>,
    down_direction: na::Unit<na::Vector3<f32>>,
    fovy: f32,
    aspect: f32,
    near: f32,
    far: f32,
}
```
That is: Saving all fields of `Camera` except those that have to be computed in the end. 

Constructing a `CameraBuilder` means inserting the default values:
```rust
impl Camera {
    fn builder() -> CameraBuilder {
        CameraBuilder {
            position: na::Vector3::new(0.0, -3.0, -3.0),
            view_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, 1.0)),
            down_direction: na::Unit::new_normalize(na::Vector3::new(0.0, 1.0, -1.0)),
            fovy: std::f32::consts::FRAC_PI_3,
            aspect: 800.0 / 600.0,
            near: 0.1,
            far: 100.0,
        }
    }
```
`CameraBuilder` gets a bunch of functions to set the values: 
```rust
    fn position(mut self, pos: na::Vector3<f32>) -> CameraBuilder {
        self.position = pos;
        self
    }
    fn fovy(mut self, fovy: f32) -> CameraBuilder {
        self.fovy = fovy.max(0.01).min(std::f32::consts::PI - 0.01);
        self
    }
    fn aspect(mut self, aspect: f32) -> CameraBuilder {
        self.aspect = aspect;
        self
    }
    fn near(mut self, near: f32) -> CameraBuilder {
        if near <= 0.0 {
            println!("setting near plane to negative value: {} — you sure?", near);
        }
        self.near = near;
        self
    }
    fn far(mut self, far: f32) -> CameraBuilder {
        if far <= 0.0 {
            println!("setting far plane to negative value: {} — you sure?", far);
        }
        self.far = far;
        self
    }
    fn view_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.view_direction = na::Unit::new_normalize(direction);
        self
    }
    fn down_direction(mut self, direction: na::Vector3<f32>) -> CameraBuilder {
        self.down_direction = na::Unit::new_normalize(direction);
        self
    }
```
All of them return a `CameraBuilder` so that they can be chained: `Camera::builder().position(/*value*/).fovy(/*value*/).view_direction(/*value*/)`.

We take directions also non-normalized (for convenience), and restrict the values of `fovy` (to a still unreasonably large range), and emit some warnings
for strange values of near and far plane. A note on these: We never want them to be equal and never want `near` equal zero. (Horrible things would
happen to the entries of our projection matrix.) 

All of the `builder` parts are finally completed with a call to `build()`. And in contrast to ash, we have no reason to implement `Deref` instead and
never use `build`. (We are not dealing with references that could lose their lifetime information, or with values that are - internally - C pointers.) 

So:
```rust
impl CameraBuilder {
    fn build(self) -> Camera {
        if self.far < self.near {
            println!(
                "far plane (at {}) closer than near plane (at {}) — is that right?",
                self.far, self.near
            );
        }
        let mut cam = Camera {
            position: self.position,
            view_direction: self.view_direction,
            down_direction: na::Unit::new_normalize(
                self.down_direction.as_ref()
                    - self
                        .down_direction
                        .as_ref()
                        .dot(self.view_direction.as_ref())
                        * self.view_direction.as_ref(),
            ),
            fovy: self.fovy,
            aspect: self.aspect,
            near: self.near,
            far: self.far,
            viewmatrix: na::Matrix4::identity(),
            projectionmatrix: na::Matrix4::identity(),
        };
        cam.update_projectionmatrix();
        cam.update_viewmatrix();
        cam
    }
```
Another sanity check on near and far plane; and a few lines ensuring that the `down_direction` is orthogonal to `view_direction` (our derivation of
the view matrix relied on that) — for the formula check the previous chapter. 

Finally, we replace our camera construction by
```rust 
    let mut camera = Camera::builder().build();
```

The camera setup can still be improved: The motion is not perfectly intuitive (but what the best effects of key presses are depends on the purpose of the application very heavily); 
we have no good way to set a `view_direction` directly (because the `down_direction` *must* remain orthogonal; we could use a similar trick as I have included in `build()`), we can only deal with the rotation; 
 we have no ways to set the perspective parameters (or, rather: we have to call `update_projectionmatrix` manually, afterwards) — but I'll leave it at
that. 

[Continue](028_Indexing.md)
