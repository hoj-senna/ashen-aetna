## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# A sphere
We have created single points, a triangle, and boxes. I'd like to move on to having a sphere. 

Spheres do not quite consist of triangles. They are not as flat. 

Well, let's start with an approximation. Something that has triangles on its surface. A cube maybe? We already have cubes, where's the fun in that? 

Let's take a regular icosahedron. 

At the beginning of `main`, we remove the whole setup with cubes: 
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let eventloop = winit::event_loop::EventLoop::new();
    let window = winit::window::Window::new(&eventloop)?;
    let mut aetna = Aetna::init(window)?;
    let mut ico = Model::cube();
    ico.insert_visibly(InstanceData {
        modelmatrix: na::Matrix4::new_scaling(0.5).into(),
        colour: [0.5, 0.0, 0.0],
    });
    ico.update_vertexbuffer(&aetna.allocator)?;
    ico.update_indexbuffer(&aetna.allocator)?;
    ico.update_instancebuffer(&aetna.allocator)?;
    aetna.models = vec![ico];
    let mut camera = Camera::builder().build();
    use winit::event::{Event, WindowEvent};
    eventloop.run(move |event, _, controlflow| match event {
```
and then we set to work and immediately replace `let mut ico = Model::cube();` by `let mut ico = Model::icosahedron();`. In `model.rs`, next to the
`cube()` function: 
```rust
impl Model<[f32; 3], InstanceData> {
    pub fn icosahedron() -> Model<[f32;3],InstanceData>{
        Model {
            vertexdata: vec![],
            indexdata: vec![],
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
```
Obviously, `vertexdata` and `indexdata` still have to be filled. 

How? Let's have a look: 
![icosahedron](https://upload.wikimedia.org/wikipedia/commons/9/9c/Icosahedron-golden-rectangles.svg)
(thanks, Wikipedia) 
If we decide that the rectangles in this icosahedron lie in the coordinate planes, the points end up in [b,a,0], [-b,a,0], [b,-a,0], [-b,-a,0]  (say,
for the dark green rectangle), with some values b\>a\>0; and similarly on the other axes: [a,0,b]  (plus sign changes; light green) and [0,b,a]
(purple, again with all possible sign combinations). We still have to find a and b; let us just set a=1 (for simplicity; we can scale the model
later). 

Then how large is b?  

It is a *regular* icosahedron, so the edges all have the same length. And that length is two, cf. the small side of the rectangles. Another edge runs
between [b,1,0] and [0,b,1], for example. Its length is 
<img src="svg/030_-00d6aa34d970c1e773f0fbe01e3b66f9f627693b36872b9f8896a0940362dce2.svg?sanitize=true?invert_in_darkmode" align="middle" width="175.34058pt" height="29.450066pt"/>, hence 
<img src="svg/030_-8cc260815b697ca7a6a83060834efd7b2096cfbf3dfbc7f88e06821255b06042.svg?sanitize=true?invert_in_darkmode" align="middle" width="174.35565pt" height="27.28537pt"/>, 
<img src="svg/030_-e057113e921fdd76427051d1ec98e8fa673b91a5d7e149f57e7d75ea22bc3bfc.svg?sanitize=true?invert_in_darkmode" align="middle" width="103.87095pt" height="27.28537pt"/>, or

<img src="svg/030_-cc0fbe6ed1c8001142ba2eb1b536c11da143bd1be5621e4ed10aa379542bfc47.svg?sanitize=true?invert_in_darkmode" align="middle" width="101.70863pt" height="34.055706pt"/>, the "golden ratio". 

So: 
```rust
        let phi = (1.0 + 5.0_f32.sqrt()) / 2.0;
        let darkgreen_front_top = [phi, -1.0, 0.0]; //0
        let darkgreen_front_bottom = [phi, 1.0, 0.0]; //1
        let darkgreen_back_top = [-phi, -1.0, 0.0]; //2
        let darkgreen_back_bottom = [-phi, 1.0, 0.0]; //3
        let lightgreen_front_right = [1.0, 0.0, -phi]; //4
        let lightgreen_front_left = [-1.0, 0.0, -phi]; //5
        let lightgreen_back_right = [1.0, 0.0, phi]; //6
        let lightgreen_back_left = [-1.0, 0.0, phi]; //7
        let purple_top_left = [0.0, -phi, -1.0]; //8
        let purple_top_right = [0.0, -phi, 1.0]; //9
        let purple_bottom_left = [0.0, phi, -1.0]; //10
        let purple_bottom_right = [0.0, phi, 1.0]; //11
```
and 
```rust
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
```

Which of these are connected? Well, we do have a picture, don't we? (It may help to check which ones are missing by counting how often each vertex is
used. Each of them should be part of five triangles. (And there should be twenty faces in total, it's an *icosa*hedron, after all.)
```rust
            indexdata: vec![
                0,9,8,//
                0,8,4,//
                0,4,1,//
                0,1,6,//
                0,6,9,//
                8,9,2,//
                8,2,5,//
                8,5,4,//
                4,5,10,//
                4,10,1,//
                1,10,11,//
                1,11,6,//
                2,3,5,//
                2,7,3,//
                2,9,7,//
                5,3,10,//
                3,11,10,//
                3,7,11,//
                6,7,9,//
                6,11,7//
            ],
```

Well, if we run the program, there is ... something. I can not so clearly recognize it as an icosahedron.

Do you remember how to use a wireframe mode?

In `renderpass_and_pipeline.rs` there is a line about the "polygon mode". Let's set it to `.polygon_mode(vk::PolygonMode::LINE);` for the time being. 

Ah, that looks better.

Now we want to get closer to a sphere. What can we do? 

We need more, smaller triangles. 

But figuring them out by hand (or by identifying points on a printout) seems soo much work. Can we automate that? Sure. 

We can consider each triangle. When given one: 


<p align="center"><img src="svg/030_-b0a9b6c2317fc8610ed12c99db715a47dd56e89ea2deb307e3f90906360adb76.svg?sanitize=true?invert_in_darkmode" align="middle" width="120.410095pt" height="96.17294pt"/></p>

we can try to split it into several. For example: 


<p align="center"><img src="svg/030_-3ff6077ff33d9b352010aaf4150442b1c007360ba6bda0da47117c7f0bf5eeb2.svg?sanitize=true?invert_in_darkmode" align="middle" width="120.410095pt" height="96.17294pt"/></p>

But then the two new triangles have a different shape than the one before, and if we repeat this too often, we could end up with many rather thin
triangles. 

Also: How do we choose which of the points becomes part of two new triangles, and which ones remain part of only one each? 

We could find answers to those questions — but let us instead split the triangle into four: 


<p align="center"><img src="svg/030_-ec7dc15bf8a0956d6fc0f228ce1bedce577c81d8f6768b81e51cf14e74bc97f7.svg?sanitize=true?invert_in_darkmode" align="middle" width="126.004486pt" height="110.04982pt"/></p>



Let us invent a function `refine` that does this to all triangles of the model. We will then call it, after creating the icosahedron and before
updating the buffers: 

```rust
    let mut ico = Model::icosahedron();
    ico.refine();
    ico.insert_visibly(//etc.
```

The function will be a new method of `Model`, let's start by this: 

```rust
    pub fn refine(&mut self){
        let mut new_indices=vec![];
        for triangle in self.indexdata.chunks(3){
            new_indices.extend_from_slice(triangle);
        }
	self.indexdata=new_indices;
    }
```
This does not yet change the model. Better: 
```rust
            let a = triangle[0];
            let b = triangle[1];
            let c = triangle[2];
            let mab =// ?  
            let mbc =// ?
            let mca =// ?
            new_indices.extend_from_slice(&[mca, a, mab, mab, b, mbc, mbc, c, mca, mab, mbc, mca]);
```
Now where do we get the midpoints `mab`, `mbc`, `mca` from? 

We compute them from the points A, B, C. Note: what we currently have are the *indices* of the points A, B, C, not the points. We'll first get those
and call them `vertex_a` etc. Then we compute the midpoints (`vertex_ab` etc.), push them onto the `vertexdata` vector and finally get their indices.

```rust
    pub fn refine(&mut self) {
        let mut new_indices = vec![];
        for triangle in self.indexdata.chunks(3) {
            let a = triangle[0];
            let b = triangle[1];
            let c = triangle[2];
            let vertex_a = self.vertexdata[a as usize];
            let vertex_b = self.vertexdata[b as usize];
            let vertex_c = self.vertexdata[c as usize];
            let vertex_ab = [
                0.5 * (vertex_a[0] + vertex_b[0]),
                0.5 * (vertex_a[1] + vertex_b[1]),
                0.5 * (vertex_a[2] + vertex_b[2]),
            ];
            let mab = self.vertexdata.len() as u32;
            self.vertexdata.push(vertex_ab);
            let vertex_bc = [
                0.5 * (vertex_b[0] + vertex_c[0]),
                0.5 * (vertex_b[1] + vertex_c[1]),
                0.5 * (vertex_b[2] + vertex_c[2]),
            ];
            let mbc = self.vertexdata.len() as u32;
            self.vertexdata.push(vertex_bc);
            let vertex_ca = [
                0.5 * (vertex_c[0] + vertex_a[0]),
                0.5 * (vertex_c[1] + vertex_a[1]),
                0.5 * (vertex_c[2] + vertex_a[2]),
            ];
            let mca = self.vertexdata.len() as u32;
            self.vertexdata.push(vertex_ca);
            new_indices.extend_from_slice(&[mca, a, mab, mab, b, mbc, mbc, c, mca, mab, mbc, mca]);
        }
        self.indexdata = new_indices;
    }
```

Okay. But each edge should be part of two triangles. Doesn't that mean we are creating every midpoint twice? It should. 
```rust
        dbg!(&self.indexdata.len());
        dbg!(&self.vertexdata.len());
```
at the end of this function yields 240 and 72. 

We'll get that down to 240 and 42. (That's the correct number, at least: 240/3 are 80 triangles, which is 4 times the 20 faces we had in the
beginning; 42 is 12 + 30 for the twelve points we started with plus one for each edge (or rather: the midpoint of each edge). How do we know the number of edges? The icosahedron is a nice convex body without holes, so the numbers V of vertices, E of edges and F of faces satisfy V - E + F =2 by Euler's polyhedron formula. And we knew V=12 and F=20.) 

Back on track: How do we get these correct numbers? 

We store which points we have already used to compute a new one and use the stored result if one exists: 
```rust
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
                let vertex_ab = [
                    0.5 * (vertex_a[0] + vertex_b[0]),
                    0.5 * (vertex_a[1] + vertex_b[1]),
                    0.5 * (vertex_a[2] + vertex_b[2]),
                ];
                let mab = self.vertexdata.len() as u32;
                self.vertexdata.push(vertex_ab);
                midpoints.insert((a, b), mab);
                midpoints.insert((b, a), mab);
                mab
            };
            let mbc = if let Some(bc) = midpoints.get(&(b, c)) {
                *bc
            } else {
                let vertex_bc = [
                    0.5 * (vertex_b[0] + vertex_c[0]),
                    0.5 * (vertex_b[1] + vertex_c[1]),
                    0.5 * (vertex_b[2] + vertex_c[2]),
                ];
                let mbc = self.vertexdata.len() as u32;
                midpoints.insert((b, c), mbc);
                midpoints.insert((c, b), mbc);
                self.vertexdata.push(vertex_bc);
                mbc
            };
            let mca = if let Some(ca) = midpoints.get(&(c, a)) {
                *ca
            } else {
                let vertex_ca = [
                    0.5 * (vertex_c[0] + vertex_a[0]),
                    0.5 * (vertex_c[1] + vertex_a[1]),
                    0.5 * (vertex_c[2] + vertex_a[2]),
                ];
                let mca = self.vertexdata.len() as u32;
                midpoints.insert((c, a), mca);
                midpoints.insert((a, c), mca);
                self.vertexdata.push(vertex_ca);
                mca
            };
            new_indices.extend_from_slice(&[mca, a, mab, mab, b, mbc, mbc, c, mca, mab, mbc, mca]);
        }
        self.indexdata = new_indices;
        dbg!(&self.indexdata.len());
        dbg!(&self.vertexdata.len());
    }
```

Better. (And the `dbg!` lines at the end of that function can go.) 

We wanted to create a sphere: 
```rust
    pub fn sphere(refinements: u32)->Model<[f32;3],InstanceData>{
        let mut model=Model::icosahedron();
        for _ in 0..refinements{
            model.refine();
        }
        model
    }
```
and 
```rust
    let mut aetna = Aetna::init(window)?;
    let mut sphere = Model::sphere(3);
    sphere.insert_visibly(InstanceData {
        modelmatrix: na::Matrix4::new_scaling(0.5).into(),
        colour: [0.5, 0.0, 0.0],
    });
    sphere.update_vertexbuffer(&aetna.allocator)?;
    sphere.update_indexbuffer(&aetna.allocator)?;
    sphere.update_instancebuffer(&aetna.allocator)?;
    aetna.models = vec![sphere];
    let mut camera = Camera::builder().build();
```

Looking at it reveals that it still is an icosahedron. Albeit one with faces split into smaller triangles. 

A sphere is defined by all vertices having the same distance to its centre. We can include that in the function: 
```rust
    pub fn sphere(refinements: u32) -> Model<[f32; 3], InstanceData> {
        let mut model = Model::icosahedron();
        for _ in 0..refinements {
            model.refine();
        }
        for v in &mut model.vertexdata {
            let l = (v[0] * v[0] + v[1] * v[1] + v[2] * v[2]).sqrt();
            *v = [v[0] / l, v[1] / l, v[2] / l];
        }
        model
    }
```

Much better. The lines coming from the back of the sphere are a bit distracting. We can remove them. 

```rust
        let rasterizer_info = vk::PipelineRasterizationStateCreateInfo::builder()
            .line_width(1.0)
            .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
            .cull_mode(vk::CullModeFlags::BACK)
            .polygon_mode(vk::PolygonMode::LINE);
```
(in `renderpass_and_pipeline.rs` in `Pipeline::init()`; we had `CullModeFlags::NONE` before)

This setting means that we only see triangles where the points (according to their order in the index buffer) appear counterclockwise on the screen.
The resulting image on screen suggests that I was lucky when numbering the points. Although mistakes would be difficult to spot for a model with such
a symmetric front and back. Also: This setting is one of the prime suspects whenever "everything should be right", but some model (parts) remain(s)
invisible.

We revert the polygon mode back to `vk::PolygonMode::FILL`. We are interested in solid spheres.

Running the program, I see a solid "circle". But the wireframe looked much more three-dimensional. 

Something is imperfect. We'll have to look into that next. 

(First a small closing note for this chapter, before we collect too many warnings again: 
```
warning: method is never used: `cube`
   --> src/model.rs:237:5
```
Let's annotate it with `#[allow(dead_code)]`.) 


[Continue](031_Normals.md)
