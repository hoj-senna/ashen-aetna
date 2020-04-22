## Ashen Aetna
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Preface 
Why another graphics programming tutorial? I don't think there are *too many* Vulkan tutorials already, especially not in Rust. In any case, a slightly
different explanation may be helpful (be it on Vulkan, on Rust, or on the basics of 3D in general). Some words on my plans with this see below; first
a few words about "the author":


What qualifies me to write such a tutorial? Approximately nothing, to be honest. I'm a hobby programmer who starts on some small programming project
when there's some free time, often during holidays, and then returns to these projects months later as time and mood permit. This has the effect that
I have forgotten a large part of my previous programming experiments again and have to start anew (to some extent). I hope that this tutorial helps me
the next time I want to get started on graphics/Vulkan. (It also serves as a higher-level substitute of commenting the code.) 

Am I at least an experienced Rust programmer? Nope, see above: hobbyist occasionally dabbling in some programming. Over the last few times the
programming language of choice has been Rust. Here my friend the compiler tells me upfront that I'm doing something stupid and refuses to compile the
program, saving me some hours of searching for a bug (that an experienced C++ programmer would claim to not having introduced in the first place)
which would have crashed the program otherwise. At the same time it's powerful and fast. I may have picked up one or two good ideas over time, but
have neither had any formal Rust education nor experience with large Rust projects or collaboration on Rust code, so the code I will write may be 
completely unidiomatic at times and may still contain bad ideas. It will probably not look too ugly, however, because at some time I have installed
rustfmt and configured it to run on every save; and for some other things Clippy might yell at me, which I also often try to avoid. I will probably
occasionally comment on some completely obvious and trivial issues (just because I wished I had known that earlier than I actually did or because it's
a new observation for me) or give misleading explanations for stuff someone else would have known better. Nevertheless, I have often found it quite
helpful to see the reasoning for some programming choices or changes explained, and maybe I hit one or two places where my reasoning helps someone out
there — and if I'm completely wrong about something or do it extremely stupidly, I'd, of course, appreciate a comment.

It looks better for the mathematics that necessarily come with "anything 3D": Elementary maths will suffice, and I am somewhat confident about those.


Some (big) thanks are in order. 

This tutorial partially owes its existence to (the creators of) Vulkan and Rust (obviously), ash, vulkan-tutorial.com, github.com/unknownue/vulkan-tutorial-rust, 
learnopengl.com, github.com/SaschaWillems/Vulkan, but also of glium and nehe.gamedev.net. 
(And probably more that I am not thinking of right now.) 

**Thank you!** 

(To be clear, they have neither endorsed this project in any way nor are they responsible for the (probably many) mistakes I will make.) 

Some parts may be better explained on one of the tutorial sites mentioned, some parts will be extremely close or could as well be copied (but I want to have the
information in one place, cf. my motivation as starting point for me when I next begin dabbling in graphics programming). 


[Continue](001_Plan.md)
