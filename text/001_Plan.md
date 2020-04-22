## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# What's the plan? — Some hints on what to expect. 

"Follow along" means: follow along how I'm *stumbling* onto this "volcano". 

The scope (indicated by the subtitle) is quite ill-defined and much too large. There is 3D graphics in general, and there are many steps between "I
know how to describe a point in space by means of coordinates", "I know what to draw on the screen" and "... so that it even looks realistic or good". 
Then there is Vulkan, which is essentially the question: How do I talk to my graphics card? — To a large extent, that's an issue rather separate from
the first.
Closely related to it, there is the question "How does this look in Rust code?" (or, here: "how do I use ash?"); there also may be other aspects of Rust or
things which to use Rust for (with a loose connection to graphics, I guess) that I might want to explore while I'm climbing "Aetna" ... 
("Stumbling" is never "moving in a straight line" only.)


This tutorial is primarily supposed to be text, supported by source code (as opposed to commented source code). The order is the temporal order "when did I
come across/have to write this thing", not necessarily the order in which the things appear in the final code. Source code will be included. Follow
along, copy and paste. Often, pieces of code will appear and be revisited several times, with small modifications (see above: temporal order). [I hope
it becomes sufficiently clear where to paste the lines.] On another note: Even while writing the first parts, I've noticed quite some instances of
"Oh, I could have done that better, in preparation of this thing I'm now dealing with". I have largely abstained from going back and "fixing" it.
(Otherwise I'd still be busy fixing things, and not moving on, and you certainly would not yet be able to read this.) I *might* later return to
earlier parts to improve them, but for now getting something written is more important.


The beginning will be heavily focused on the Vulkan aspect ("dealing with the API", not so much "learning about graphics") — we have to set something
up first, before we can use it in more interesting applications, but I hope that I eventually actually get to cover some more interesting "graphics
stuff". 
It is at least equally likely that this "tutorial" will never reach this point, but instead will be abandoned after a few initial chapters. I make
no promises. Writing takes time (even more than I anticipated), the so-called "real life" may interfere, and so on. I would sincerely hope that even
what exists up to that point might be helpful to someone. 
Especially with the beginning, it is quite possible that you will not notice much that is new if you have already read one of the other Vulkan
tutorials.

I hope to go into some details about 3D graphics basics, even though I would not need these were I to write the tutorials for myself only, and
although I am not entirely sure how many beginners to that field start with Vulkan. (Especially if you are new to these things, be advised that you do
not have to read the chapters in order and that it may be a good idea to only skim the Vulkan setup part, which will take some time. (Grab the code from 
its end and return later to read the details.))

The difficulty level will vary, as will the focus of my explanations, and I have no uniform list of prerequisites that I can give to you. (I'll try to
aim at "complete beginner" with both Vulkan and 3D graphics in general. — At least that's the plan for now, but it may change.)

Nevertheless: Have fun! and: [Let's get started](002_Beginnings.md)!

(You do seem to have quite some patience if you have read my ramblings up to this point.) 
