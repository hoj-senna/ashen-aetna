## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# The first triangle

### PipelineInputAssembly and PrimitiveTopology

By now, we are drawing six separate points. It is more usual to want to draw solid objects than single, isolated points. What we can see of a solid
object is its surface. 

Every surface is flat. Okay, that's approximately as true as the statement that every function is linear: Convenient fiction because flat surfaces or
linear maps are relatively easy to deal with, but fiction nonetheless. And, more importantly: Approximately true if we can "look from infinitely
close". 

One point alone is not enough to describe a flat surface, a plane, nor are two points. Three points, on the other hand, ... 

And we can split every (flat) surface into triangles. Therefore, essentially all geometry is represented as collection of triangles. 

We should tell our program that we want to draw triangles, that is: to interpret three vertices taken together as one triangle. In other words: We
want to change how our input is *assembled* into geometric objects. In our `Pipeline::init()` function we are currently setting the "input assembly
state" as `POINT_LIST`. Let us change this to triangles (`TRIANGLE_LIST`): 

```rust
        let input_assembly_info = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::TRIANGLE_LIST);
```

And ... that's it. More changes to the code (from the end of last chapter) are not needed: We obtain a nice green triangle.

Let us add more points. After the previous chapter we know how to do this: Just pour some more data into the buffers and adjust the number of vertices
in the draw call. 

Buffers: 
```rust
        let buffer1 = Buffer::new(
            &allocator,
            96,
            vk::BufferUsageFlags::VERTEX_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer1.fill(
            &allocator,
            &[
                0.5f32, 0.0f32, 0.0f32, 1.0f32, 0.0f32, 0.2f32, 0.0f32, 1.0f32, -0.5f32, 0.0f32,
                0.0f32, 1.0f32, -0.9f32, -0.9f32, 0.0f32, 1.0f32, 0.3f32, -0.8f32, 0.0f32, 1.0f32,
                0.0f32, -0.6f32, 0.0f32, 1.0f32,
            ],
        )?;
        let buffer2 = Buffer::new(
            &allocator,
            120,
            vk::BufferUsageFlags::VERTEX_BUFFER,
            vk_mem::MemoryUsage::CpuToGpu,
        )?;
        buffer2.fill(
            &allocator,
            &[
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32,
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32,
                1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32, 1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32,
            ],
        )?;
```

Draw call: 
```rust
            logical_device.cmd_draw(commandbuffer, 6, 1, 0, 0);
```

And we have two triangles.

By the way, what would happen if we only sent five vertices and not six? Let's try (no need to adjust also the buffers): 
```rust
            logical_device.cmd_draw(commandbuffer, 5, 1, 0, 0);
```
Back to one triangle. That makes sense: The first three points form the first triangle. The remaining two points ... well, do not form a triangle, so
they are ignored. Let's switch the number back to `6`. and move on. 

There are other types of "topology" than a list of points or a list of triangles. For example, we could use a list of lines (taking two vertices
each). Thinking of lines, if we used lines and sent points P1, P2, P3, P4, P5, P6, we'd get a line between P1 and P2, another one between P3 and P4
and one between P5 and P6. (Actually, you can try this: Just switch the line about input topology to `.topology(vk::PrimitiveTopology::LINE_LIST);`).
It is easy to imagine that we might want to connect P1 to P2 to P3 and so on, without gaps. That is also possible: `vk::PrimitiveTopology::LINE_LIST`.
Similarly, there is a variant for triangles reusing some of the previous points. For an overview with pictures, you can consult the subsections of 
[Vulkan spec: Primitive Topologies](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#drawing-primitive-topologies). (Some
of them make no sense for us now, because we are not using a geometry shader.) 

When you're done playing around with these settings, switch the code back to `TRIANGLE_LIST`. 

### PolygonFillMode and DeviceFeatures

It could, nevertheless, occasionally be useful to switch to a "wireframe mode" and to only show the outlines of triangles. Exchanging the
`TRIANGLE_LIST` for a `LINE_LIST` as in the previous paragraph would not be a good fit for that: One triangle would correspond to one and a half lines
(`LINE_LIST`) or there would be connections between the third vertex of each triangle and the first of the next one (`LINE_STRIP`). Rearranging the
list of vertices for this change (maybe for a quick debugging check)? Not feasible. 
So, keeping the triangle list, can we do something? 

Yes, else I wouldn't have raised the point here, probably.

Also on pipeline initialization, in our old code we find the following settings for the rasterizer stage: 
```rust
        let rasterizer_info = vk::PipelineRasterizationStateCreateInfo::builder()
            .line_width(1.0)
            .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
            .cull_mode(vk::CullModeFlags::NONE)
            .polygon_mode(vk::PolygonMode::FILL);
```
In the last line herein, we can replace `PolygonMode::FILL` by `PolygonMode::LINE` (or even `PolygonMode::POINT`). Then the triangles are drawn as
lines or points only. They are still triangles. (You can check this by drawing only 5 or 2 instead of the 6 points. You will notice that the whole
incomplete triangle is not drawn, even if the data for one or two of its points were provided.) 

I said "then the triangles are drawn as" — but actually something else is happening at the same time: The validation layers complain: 
```
[Debug][error][validation] "vkCreateGraphicsPipelines parameter, VkPolygonMode pCreateInfos->pRasterizationState->polygonMode cannot 
be VK_POLYGON_MODE_POINT or VK_POLYGON_MODE_LINE if VkPhysicalDeviceFeatures->fillModeNonSolid is false."
```
At least on my machine, it still works. But we should take care of this (in fact: of every) error that is reported.

What's behind this one? 

There are some special features that not every GPU has and that should not be taken for granted. Let's not put too much emphasis on "special" here. A
few of them seem rather normal. But they have to be explicitly enabled. (I think it's one of the core design tenets of Vulkan: Make as much explicit as possible.)

Let us first find out about these features. I'll adapt the function listing and choosing the physical device first, so that we can call it by 
```rust
        let (physical_device, physical_device_properties, physical_device_features) =
            init_physical_device_and_properties(&instance)?;
```
Getting them means calling the corresponding method of `instance`: 
```rust
fn init_physical_device_and_properties(
    instance: &ash::Instance,
) -> Result<
    (
        vk::PhysicalDevice,
        vk::PhysicalDeviceProperties,
        vk::PhysicalDeviceFeatures,
    ),
    vk::Result,
> {
    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    let mut chosen = None;
    for p in phys_devs {
        let properties = unsafe { instance.get_physical_device_properties(p) };
        let features = unsafe { instance.get_physical_device_features(p) };
        if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
            chosen = Some((p, properties, features));
        }
    }
    Ok(chosen.unwrap())
}
```
It would be very reasonable to take these features into account when choosing which device to use (if this were code for production use and not just
tutorial code). I'll at least save the result in our big `Aetna` struct. But I'm not reacting to it anywhere (like testing if drawing lines only is
possible at all when setting up for pipeline creation — which, again, would probably be the thing to do in a real application). 

We still have to request the feature. This happens on creation of the *logical* device, i.e. in `fn init_device_and_queues`.

Here it is part of `DeviceCreateInfo`: 
```rust
    let device_create_info = vk::DeviceCreateInfo::builder()
        .queue_create_infos(&queue_infos)
        .enabled_extension_names(&device_extension_name_pointers)
        .enabled_layer_names(&layer_name_pointers)
        .enabled_features(&features);
```
where we have chosen the features to use as follows: 
```rust
    let features = vk::PhysicalDeviceFeatures::builder().fill_mode_non_solid(true);
```
And then we can watch our triangles as lines or points only, again. This time without error message.

By the way, if we request some feature that our GPU does not have, the program crashes, telling us: 
```
Error: ERROR_FEATURE_NOT_PRESENT
```

Okay, back to `PolygonMode::FILL`.

### What happens on the way from vertex to fragment shader? 

At the moment, we are filling our size-and-colour vertexbuffer with the following values: 
```rust
        buffer2.fill(
            &allocator,
            &[
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32,
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32,
                1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32, 1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32,
            ],
        )?;
```
in shorter: the first triangle green, the second yellow. 

And while we are passing in the colours of the single points, the whole triangle ends up in this colour. (Which is as it should be: We have picked
`PolygonMode::FILL`, after all.) 

But what happens (which colour is chosen) if not all vertices of a triangle have the same colour? 

Let's find out. Here, I set the very last point to blue instead of yellow.
```rust
        buffer2.fill(
            &allocator,
            &[
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32,
                15.0f32, 0.0f32, 1.0f32, 0.0f32, 1.0f32, 1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32,
                1.0f32, 0.8f32, 0.7f32, 0.0f32, 1.0f32, 1.0f32, 0.0f32, 0.0f32, 1.0f32, 1.0f32,
            ],
        )?;
```
How does it look? 

The colour changes relatively smoothly from yellow at the first (and second) vertex to blue at the third.

We learn that the colour value, the value we pass from vertex to fragment shader, is interpolated, and points/pixels/fragments somewhere in the middle of the triangle
do not receive the colour value of one of the points, but a combination of the values from all the points, according to their position. 

If you think about it: That's the same thing that's also happening to the position variable, isn't it? 

And usually, it's exactly what we want. 

In cases where it is not the desired effect, we can turn this behaviour off: 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=0) flat in vec4 data_from_the_vertexshader;

void main(){
	theColour= data_from_the_vertexshader;
}
```

Have you spotted the difference to before? It's small: `flat` is the keyword that makes the whole triangle use the value from one of the vertices.
Which vertex? The first one. (The "provoking vertex"; it's marked in the diagrams on the [page](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#drawing-primitive-topologies) on primitive topologies.)

For now, let's remove `flat` again.

I think we should look at more than a triangle. Maybe a box first? 

[Continue](021_Boxes.md)

