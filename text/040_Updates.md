## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Cleanup and updates

It has been some time since the last update (and I don't mean the slowness of the chapters, but the update of the Rust version on my computer). 


```
rustup default 1.47.0
```
(Why not `rustup update stable`? We'll get to that. Today this would, by the way, give Rust 1.48.0.) 

After a bit of time for recompiling, the program still runs. Good. 

Now let's try the actually newest version: 

```
rustup update stable
rustup default stable
```
(Now it's set to the newest (stable) Rust, that is, 1.48.0.) 

Let's compile again: 
```
thread 'main' panicked at 'attempted to zero-initialize type `ash::Device`, which is invalid', /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/mem/mod.rs:622:9
```
Ouch. 

This is some problem occurring within `vk-mem`. (And thanks go to xla for noticing and pointing out this [issue](https://github.com/hoj-senna/ashen-aetna/issues/2).) This is fixed in a newer version of `vk-mem`. So, apparently, an update of `Cargo.toml` is in order: 


```
vk-mem = "0.2.3"
```

Oops: 
```
error: failed to select a version for the requirement `vk-mem = "^0.2.3"`
candidate versions found which didn't match: 0.2.2, 0.2.0, 0.1.9, ...
location searched: crates.io index
```
The fix is not yet released. Well, it already exists in the repository. Let's point cargo there: 
```
vk-mem = { git = "https://github.com/gwihlidal/vk-mem-rs", version = "0.2.3" }
```

... and ... 

More errors:
```
error[E0277]: the `?` operator can only be applied to values that implement `Try`
```
`vk-mem`'s changelog tells us the reason: 
```
    Removed Result return values from functions that always returned Ok(())
```
That means: We can get rid of some question marks and some `.expect`s. Let's do so. 

Okay, that makes these errors disappear. But: 
```
error: linking with `cc` failed: exit code: 1
```

Oooookay.

I think I now know better what "but master doesn't seem to be building right now" in the response to [vk-mem's corresponding issue](https://github.com/gwihlidal/vk-mem-rs/issues/42) means. Next step: Downloading this repository and trying locally. In `Cargo.toml` that means something like
```
vk-mem = { path = "../../vk-mem-rs-master/", version = "0.2.3" }
```
with the path relative to `Cargo.toml` (and I must not forget to download the included `vendor` folder separately, as this is a different repository).

Of course, the error remains. 

A bit of poking around (and thoughts of giving up -- after all, I don't really want to deal with C++ and linker errors) and helpful ... pointers ... by
the [VMA documentation](https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/configuration.html#config_Vulkan_functions), 
in line 4021 of `vk_mem_alloc.h` (in the CONFIGURATION SECTION) I write  
```cpp
    #define VMA_DYNAMIC_VULKAN_FUNCTIONS 0
```
(instead of `1`).  

And now, magically, it works again. (No problems with dynamically loading functions, if we decide not to load functions dynamically, I suppose.) 
I don't know if this is "the right solution" or "a dirty hack", but I don't really care that much. (I just hope for a fix in the next version of
vk-mem.) 




There are also new versions of the crates my `Cargo.toml` lists. For example, 

```
ash = "0.31.0"
```
Oh: 

```
   --> src/main.rs:361:16
    |
361 |         .image(destination_image)
    |                ^^^^^^^^^^^^^^^^^ expected struct `ash::vk::Image`, found a different struct `ash::vk::Image`
    |
    = note: perhaps two different versions of crate `ash` are being used?
```

I really love how the compiler directly tells me what the problem might be. And of course it is correct. `cargo tree` reveals that, inter alia, 
```
├── vk-mem v0.2.3 (/home/joh/prog/foreign-rust/vk-mem-rs-master)
│   ├── ash v0.30.0
```
Well, vk-mem's `Cargo.toml` only says
```
ash = ">= 0.27.1"
```
Actually, I'm a bit surprised that cargo does not resolve this (at least after some `cargo clean`); but I do have access to vk-mem's `Cargo.toml`
after having downloaded this repository anyway. Thus, it's simple to change this to `"= 0.31.0"` (and possibly change it back after recompiling once
...) 

More updates: 
```
vk-shader-macros = "0.2.2"
```
could, by now, be "0.2.6". (And could have been, for several months.) Nothing breaks. That's how I like updates. 

Moreover, 
```
fontdue = "0.2.4"
```
can be updated to `"0.4.0"`. Haven't I added that only last chapter? (Well, somewhere it must be visible that I had started writing that chapter quite
some time before finishing and uploading it. In my defense: In between, I moved to a different city and obtained a new position at a different
employer.)

``` 
error[E0061]: this function takes 1 argument but 0 arguments were supplied
   --> src/text.rs:163:26
    |
163 |         let mut layout = fontdue::layout::Layout::new();
    |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^-- supplied 0 arguments
    |                          |
    |                          expected 1 argument

error[E0599]: no method named `layout_horizontal` found for struct `fontdue::layout::Layout<_>` in the current scope
   --> src/text.rs:168:16
    |
168 |         layout.layout_horizontal(&self.fonts, styles, &settings, &mut output);
    |                ^^^^^^^^^^^^^^^^^ method not found in `fontdue::layout::Layout<_>`

error: aborting due to 2 previous errors; 5 warnings emitted
```
Okay, some changes to layout. That means, `create_letters` should look a bit differently: 
```rust
    pub fn create_letters(
        &self,
        styles: &[&fontdue::layout::TextStyle],
        colour: [f32; 3],
    ) -> Vec<Letter> {
        let mut layout =
            fontdue::layout::Layout::new(fontdue::layout::CoordinateSystem::PositiveYUp);
        let settings = fontdue::layout::LayoutSettings {
            ..fontdue::layout::LayoutSettings::default()
        };
        layout.reset(&settings);
        for style in styles {
            layout.append(&self.fonts, style);
        }
        let output = layout.glyphs();
        let mut letters: Vec<Letter> = vec![];
        for glyph in output {
            letters.push(Letter {
                colour,
                position_and_shape: glyph.clone(),
            });
        }
        letters
    }
```
That is: `Layout::new` receives some information on the direction of the y-axis. (Here I use what was the default in the previous version. Changes
here should be accompanied by changes in the translation to positions in `create_vertexdata`.) And the previous 
```rust
         layout.layout_horizontal(&self.fonts, styles, &settings, &mut output);
```
is changed: We first append the styles (each one separately) to the layout and finally receive the glyphs from `layout.glyphs` and not in a `output`
variable that we firstly passed to the function. Better this way. 

But what's that?
```
thread 'main' panicked at 'creating vkImage for texture: Error { kind: Vulkan(ERROR_VALIDATION_FAILED_EXT) }', src/text.rs:398:14
```
This is in `TextTexture::from_u8s`. Printing the function arguments (by 
```rust
dbg!(&data);
dbg!(&width);
dbg!(&height);
```
) makes me suspect that the spaces are the problem: they now are a of size 0, not of size 1.

Ah, well, some new first lines for this function: 
```rust
        if data.len() == 0 {
            return Err(Box::new(TextError::EmptyData));
        }
```
where 
```rust
#[derive(Debug)]
pub enum TextError {
    EmptyData,
}
impl std::fmt::Display for TextError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        match self {
            TextError::EmptyData => {
                write!(f, "Empty Data");
            }
        }
        Ok(())
    }
}
impl std::error::Error for TextError {}
```

And I am rewarded with:
```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: EmptyData', src/text.rs:215:22
```
In that place we have 
```rust
                id = self
                    .new_texture_from_u8s(
                        &bitmap,
                        metrics.width as u32,
                        metrics.height as u32,
                        device,
                        allocator,
                        commandpool_graphics,
                        graphics_queue,
                    )
                    .unwrap() as u32;
```
Maybe rather something like this: 
```rust
                id = match self.new_texture_from_u8s(
                    &bitmap,
                    metrics.width as u32,
                    metrics.height as u32,
                    device,
                    allocator,
                    commandpool_graphics,
                    graphics_queue,
                ) {
                    Ok(the_id) => the_id as u32,
                    Err(e) => {
                        if let Some(err) = e.downcast_ref::<TextError>() {
                            match err {
                                TextError::EmptyData => {
                                    continue;
                                }
                            }
                        } else {
                            panic!();
                        }
                    }
                };
```
If there is an error, we check whether it's a case of empty data (downcasting the `Box`ed trait object), which means: skipping this letter — or
something else (no idea what), which results in a panic. That sounds reasonable to me. 


Next crate to update: `image`, from `0.23.4` to `0.23.12`. — — — No trouble here.

Next: `nalgebra` from `0.18.0` to `0.24.0`. Again, no problems.

And then there is `winit`. `0.22.0` to `0.24.0`. And once more, I spot no problem.



Enough of updates. But while I'm at it: There are a few messages "unused `std::result::Result` that must be used" that I receive. And since I wanted
to document every change I make to the code, let me briefly mention it here: I'm going to add some question marks and `.expect()`s ... 

Also, there seems to be one unnecessary `unsafe` block. I'll remove it. 

`AllText::clear_pipeline` does not need its `Allocator` argument any more. (And removed it is.) 

```
warning: use of deprecated associated function `image::DynamicImage::to_rgba`: replaced by `to_rgba8`
   --> src/main.rs:549:64
    |
549 |     let screen_image = image::DynamicImage::ImageBgra8(screen).to_rgba();
    |                                                                ^^^^^^^
    |
    = note: `#[warn(deprecated)]` on by default

warning: use of deprecated associated function `image::DynamicImage::to_rgba`: replaced by `to_rgba8`
  --> src/texture.rs:24:28
   |
24 |             .map(|img| img.to_rgba())
   |                            ^^^^^^^
```
Did I say "no trouble here"? That was the point where I should have noticed and fixed these. Well, it definitely is no trouble. I'll insert these
`8`s.

One `mut` is unnecessary. (Easy to fix. And don't worry if you don't find it. It is a leftover from one of the previous experiments and could have
been removed some time ago.)


And finally, there are a lot of "never read" fields, "unused imports" and "never used" functions. I do not want to remove these, as they could be
helpful again: It is quite possible that I will want to have spheres or lights again, at some time. 

In order to make the list of warnings shorter, I'll therefore include some crate-level 
```rust
#![allow(unused)]
```

The remaining warnings are some for `vk-mem`, and I don't intend to look at them more closely. 

[Continue](041_DebugPrintf.md)



