## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Beginnings

The "Hello World" of graphics programming is getting a triangle on the screen. 
Vulkan has made one very important design decision which is somewhat unfortunate for us in achieving this: Vulkan wants to be given information up
front. That means that a lot has to be prepared or described before the first triangle becomes visible. It means that the first steps are the most
difficult and tedious steps. (Unfortunate for someone who is beginning to learn Vulkan, but not so bad (or even: rather good) afterwards. - Be
prepared to spend a lot of time seemingly making no progress, until the triangle is there. But then the worst part is done.)

- If you feel you're getting lost in details (which would be quite natural, in my opinion), feel free to temporarily skip ahead to [Chapter 12](012_Looking_back.md) at any time, 
for a higher-level overview (or, by then, look back) of what we will have had to do to get there.

- If you do not care that much about Vulkan specifically and are more interested in more general computer graphics, jump to [Chapter 13](013_Coordinates.md)
 (This could be a good idea if you are completely new to this topic.)

Let's begin.

What do we need for a triangle? Well, we have decided that we want to need Vulkan (so we will create a "Vulkan instance") and maybe a GPU (what's the
point of graphics programming if we don't have one?). Actually, for the GPU we will wait a few pages and another step in between. 
So, first the Vulkan instance, which ash provides us with. (Ash wraps the Vulkan API (which is C code) in a very thin layer of Rust code, so that we 
can call the functions from Rust. This is a somewhat minimal amount of work and provides little additional convenience, but it means that calls to ash 
mirror the corresponding calls we might find in Vulkan tutorials written for other languages or in the Vulkan spec rather closely. 
We start a new project (cargo new vulkanrenderer --bin), and include ash in the Cargo.toml (ash="0.30.0"). We then create an entry (the, well, entry
to all things Vulkan - or: the thing that loads the dynamic library containing the volcano) and an instance: 
```rust
fn main() {
    let entry = ash::Entry::new();
    let instance = unsafe { entry.create_instance(&Default::default(), None) };
}
```
What's with the unsafe? Isn't unsafe bad? Well, no. Unsafe means that the Rust compiler does not give any guarantees about the code in here. And here,
of course, it can't: After all, we're using some external (C++) library.  Rule of thumb: There will be an "unsafe" whenever we actually call Vulkan
functions. Next problem: The entry creation can fail (therefore, entry is actually a Result, and we cannot call `entry.create_instance`), and so can
instance creation; we better adjust our main(): 

```rust
(fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let instance = unsafe { entry.create_instance(&Default::default(), None)? };
    Ok(())
}
```

We could, of course, also leave out the "`Ok(())`" and keep "`fn main(){`" without the fancier return type if we write "`.expect("something went wrong
with the entry creation")`" (and some analogue for the instance) in place of "`?`".)

This still does not work, because ash does not yet know which version of the Vulkan library to load. We tell it by including a use statement:
```rust
use ash::version::EntryV1_0;
```
(If we wanted to use Vulkan version 1.1, we'd need ``ash::version::EntryV1_1`` instead, etc.) 
In the same way, we deal with the instance: "`use ash::version::InstanceV1_0`".

We have another problem to take care of: Cleanup. More precisely: Cleanup on the Vulkan side. We have created an instance, now we have to destroy it. 

```rust
use ash::version::EntryV1_0;
use ash::version::InstanceV1_0;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let entry = ash::Entry::new()?;
    let instance = unsafe { entry.create_instance(&Default::default(), None)? };
    unsafe { instance.destroy_instance(None) };
    Ok(())
}
```

We haven't talked about the arguments of the instance creation and destruction functions. The "None" in both has to do with memory (de)allocation. If
we wanted to implement custom allocation strategies, we could provide a function callback here.
More interesting for us is the "`&Default::default()`" that I have used for the sake of brevity. Whenever we create anything in Vulkan, there may be
some customization options. When buying a car, you might want to decide on a colour. Not a Vulkan example, oh, well. With Vulkan, you write a list in
advance (in the car example: "`colour: red, number_of_wheels: 4, ...`") and pass it to the creation function. This "list" takes the form of a certain
struct, here it would be a (reference to) some ash::vk::InstanceCreateInfo. Ash is so nice to have a default implementation for all of these types.
Usually, we will not be able to just use the default as the whole argument like we did here (and it may be good for us to see which options we have to
decide about). But there are always some fields in that struct that can be filled automatically, and using Default is perfect for that. In fact, we
will have to have a look at the InstanceCreateInfo very soon anyway, so let's return to our program, which by now compiles and runs and does nothing
visible. Exciting program. But at least we have a Vulkan instance. 

[Continue](003_Validation_layers.md)
