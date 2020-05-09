## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Indexed Drawing
How many vertices does a cube have? 

8? 

Count the entries in `vertexdata`: 
```rust
    fn cube() -> Model<[f32; 3], InstanceData> {
        let lbf = [-1.0, 1.0, 0.0]; //lbf: left-bottom-front
        let lbb = [-1.0, 1.0, 1.0];
        let ltf = [-1.0, -1.0, 0.0];
        let ltb = [-1.0, -1.0, 1.0];
        let rbf = [1.0, 1.0, 0.0];
        let rbb = [1.0, 1.0, 1.0];
        let rtf = [1.0, -1.0, 0.0];
        let rtb = [1.0, -1.0, 1.0];
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
```
There are 36. On the other hand, those 36 are only 8 different points. 

We currently put 36\*3 floats into the vertex buffer. 

Could we get away with only 8\*3 floats? No, of course not: `vertexdata` also contains the information which of the points form a triangle. 

But we do note have to keep these quite different sets of information together. We can submit the position information first, and the connectivity
information separately, as a list of indices ("first, third and eigth vertex form a triangle"). That would mean 8\*3 `f32`s plus 36\*1 `u32`s. In total 60 variables instead of the 36\*3=108 we had before. 

While that may not be a tremendously huge win, remember that we are talking about a box with 8 vertices in total. For larger models, this should
really pay off.

Vulkan supports this "indexed drawing", we just have to use `.cmd_draw_indexed()` instead of `.cmd_draw()`. But let's not get ahead of ourselves, that
does require some changes elsewhere. 

For example, we have to rework our `Model` struct. In the cube example above, we want 
```rust
vertexdata:vec![lbf,lbb,ltf,ltb,rbf,rbb,rtf,rtb],
```
and nothing else. We then need a place to store those indices saying which vertices form triangles. 

(Now it's a bit *unfortunate* that I have already used "index" to refer to indices of elements in the vector of instances, but we'll cope with the
same name for slightly different types of ... indices.)

Say, 
```rust
indexdata: vec![
                0, 1, 5, 0, 5, 4, //bottom
                2, 7, 3, 2, 6, 7, //top
                0, 6, 2, 0, 4, 6, //front
                1, 3, 7, 1, 7, 5, //back
                0, 2, 1, 1, 2, 3, //left
                4, 5, 6, 5, 7, 6, //right
		]
```
(after replacing "lbf" by "0" etc.) and an 
```rust
indexbuffer: None
```
or, in full (with an additional small correction in the z-coordinates):
```rust
impl Model<[f32; 3], InstanceData> {
    fn cube() -> Model<[f32; 3], InstanceData> {
        let lbf = [-1.0, 1.0, -1.0]; //lbf: left-bottom-front
        let lbb = [-1.0, 1.0, 1.0];
        let ltf = [-1.0, -1.0, -1.0];
        let ltb = [-1.0, -1.0, 1.0];
        let rbf = [1.0, 1.0, -1.0];
        let rbb = [1.0, 1.0, 1.0];
        let rtf = [1.0, -1.0, -1.0];
        let rtb = [1.0, -1.0, 1.0];
        Model {
            vertexdata: vec![lbf, lbb, ltf, ltb, rbf, rbb, rtf, rtb],
            indexdata: vec![
                0, 1, 5, 0, 5, 4, //bottom
                2, 7, 3, 2, 6, 7, //top
                0, 6, 2, 0, 4, 6, //front
                1, 3, 7, 1, 7, 5, //back
                0, 2, 1, 1, 2, 3, //left
                4, 5, 6, 5, 7, 6, //right
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
}
```
together with, of course, a changed `Model`:
```rust
struct Model<V, I> {
    vertexdata: Vec<V>,
    indexdata: Vec<u32>,
    handle_to_index: std::collections::HashMap<usize, usize>,
    handles: Vec<usize>,
    instances: Vec<I>,
    first_invisible: usize,
    next_handle: usize,
    vertexbuffer: Option<Buffer>,
    indexbuffer: Option<Buffer>,
    instancebuffer: Option<Buffer>,
}
```
The following is an adjusted version of `Model::update_vertexbuffer`:
```rust
fn update_indexbuffer(
        &mut self,
        allocator: &vk_mem::Allocator,
    ) -> Result<(), vk_mem::error::Error> {
        if let Some(buffer) = &mut self.indexbuffer {
            buffer.fill(allocator, &self.indexdata)?;
            Ok(())
        } else {
            let bytes = (self.indexdata.len() * std::mem::size_of::<u32>()) as u64;
            let mut buffer = Buffer::new(
                &allocator,
                bytes,
                vk::BufferUsageFlags::INDEX_BUFFER,
                vk_mem::MemoryUsage::CpuToGpu,
            )?;
            buffer.fill(allocator, &self.indexdata)?;
            self.indexbuffer = Some(buffer);
            Ok(())
        }
    }
```
In some places "vertex" has been replaced by "index", and `size_of` now refers to `u32` instead of `V`. We could just replace it by `4`, actually. 

One important change: The `BufferUsageFlags` have also turned into `INDEX_BUFFER` (from `VERTEX_BUFFER`). 

We also make sure to call this function right after `update_vertexbuffer`:
```rust
    cube.update_indexbuffer(&aetna.allocator);
```

In the `draw` function, we first bind the index buffer: 
```rust
                          logical_device.cmd_bind_index_buffer(
                                commandbuffer,
                                indexbuffer.buffer,
                                0,
                                vk::IndexType::UINT32,
                            );
```
We indicate the buffer, an offset (in case we don't want to use the first entries of this buffer) and the type of the indices. We have stored `u32`s,
so `vk::IndexType::UINT32` it is; but we could also use `u16` (which would certainly be enough for our cube) or even `u8`, although that requires
another extension. 

The draw command is changed to `draw_indexed`:
```rust
                            logical_device.cmd_draw_indexed(
                                commandbuffer,
                                self.indexdata.len() as u32,
                                self.first_invisible as u32,
                                0,
                                0,
                                0,
                            );
```
Now the amount of vertices to draw (per instance) is no longer given by the number of entries in `vertexdata`, but by the number of entries in
`indexdata`. We still need the amount of instances. Then comes the first index to access in the index buffer, a vertex offset (for example `-1` if you
want every entry `1` in the index buffer to refer to the first vertex in the vertex buffer, i.e. that at position `0`), and the instance ID of the
first instance to draw.

The draw function in total looks like this (only the line `if let Some(indexbuffer) = &self.indexbuffer` was not yet commented on): 
```rust
    fn draw(&self, logical_device: &ash::Device, commandbuffer: vk::CommandBuffer) {
        if let Some(vertexbuffer) = &self.vertexbuffer {
            if let Some(indexbuffer) = &self.indexbuffer {
                if let Some(instancebuffer) = &self.instancebuffer {
                    if self.first_invisible > 0 {
                        unsafe {
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                0,
                                &[vertexbuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_vertex_buffers(
                                commandbuffer,
                                1,
                                &[instancebuffer.buffer],
                                &[0],
                            );
                            logical_device.cmd_bind_index_buffer(
                                commandbuffer,
                                indexbuffer.buffer,
                                0,
                                vk::IndexType::UINT32,
                            );
                            logical_device.cmd_draw_indexed(
                                commandbuffer,
                                self.indexdata.len() as u32,
                                self.first_invisible as u32,
                                0,
                                0,
                                0,
                            );
                        }
                    }
                }
            }
        }
    }
```
Finally, some cleanup for `Aetna::drop`:
```rust
                if let Some(ib) = &m.indexbuffer {
                    self.allocator
                        .destroy_buffer(ib.buffer, &ib.allocation)
                        .expect("problem with buffer destruction");
                }
```

If we run the program, everything looks as before. Which is as it should be.


[Continue](029_Cleanup.md)
