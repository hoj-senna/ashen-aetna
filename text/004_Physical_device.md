## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Physical device 

Next step: Is there a graphics card we can use? Or, rather, a "physical device"? Let us look for some: 
```rust
    let phys_devs = unsafe { instance.enumerate_physical_devices()? };
    for p in phys_devs {
        let props = unsafe { instance.get_physical_device_properties(p) };
        dbg!(props);
    }
```
That should give a list in the terminal, including the properties of the devices that we might wish to use. Of course, we now have to choose one. That
could be the first one in this Vec. (That's reasonable in tutorials. I would, however, recommend to check at least once by some dbg! whether it's the
right choice - or if there might be a different integrated graphics card in your computer that you had forgotten about. Another choice might be
picking the one with the right name: 
```rust
     let (physical_device, physical_device_properties) = {
        let mut chosen = None;
        for p in phys_devs {
            let properties = unsafe { instance.get_physical_device_properties(p) };

            let name = String::from(
                unsafe { std::ffi::CStr::from_ptr(properties.device_name.as_ptr()) }
                    .to_str()
                    .unwrap(),
            );
            if name == "GeForce GTX 760" {
                chosen = Some((p, properties));
            }
        }
        chosen.unwrap()
    };
```
I guess that also tells what graphics card my computer contains... Needless to say, this way is not a good choice if you ever want to run the
program on a different computer. For the moment, let's pick any (the last listed) discrete GPU. 
```rust
    let (physical_device, physical_device_properties) = {
        let mut chosen = None;
        for p in phys_devs {
            let properties = unsafe { instance.get_physical_device_properties(p) };
            if properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
                chosen = Some((p, properties));
            }
        }
        chosen.unwrap()
    };
```
Still, might not be optimal (it's a way to exclude my laptop, for example), but is easy enough to adjust. 
Now that we have a physical device, we will need queue families on it.

[Continue](005_Queues.md)
