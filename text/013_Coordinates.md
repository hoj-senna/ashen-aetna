## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Coordinates 
*(If you haven't followed along with the Vulkan setup, because your priority lies with graphics programming and not with API-specifics of talking to the
driver, you should grab the file contents near the end of [Chapter 11](011_Presentation_synchronisation.md). You may (in case of Windows or Mac: will) 
have to adjust some platform-specific code; it should be confined to [Chapter 6](006_Window.md). Some problems could also result from the setup in [Chapter
4](004_Physical_device.md))*

Now that we have something on the screen, let us have a closer look at the programs running on the GPU, the shaders. 
You will recall our vertex shader: 
```glsl
	#version 450

	void main() {
	    gl_PointSize=2.0;
	    gl_Position = vec4(0.0,0.0,0.0,1.0);
	}
```
Let us try to see whether it really has an effect. Maybe we can change the point size for a quick start: 
```glsl
	    gl_PointSize=200.0;
```
(Run the program.) 
Whoa, that's a big point!
(Maybe we return it to a more reasonable value. — I'll use 10.0, so that it's still easy to spot.) 

More interestingly: The coordinates of the point. (Four numbers, the fourth of which we are ignoring for now. It is set to 1.0, and I have decided to
treat it as nothing more than a marker that this is a point, at this point. We'll talk about the first three numbers, which I'll (very creatively)
call x, y and z.) 

Currently, the point is located in the middle of the window. Try 
```glsl
	    gl_Position = vec4(1.0,0.0,0.0,1.0);
```
It should end up on the right border of the window, at the same height as before. 

Similarly, 

```glsl
	    gl_Position = vec4(-1.0,0.0,0.0,1.0);
```
lies on the left. 

Note: The first coordinate, x, ranges from -1 on the left to 1 on the right of the window (more accurately, of the viewport). 
Next component, y. 

```glsl
	    gl_Position = vec4(0.0,-1.0,0.0,1.0);
```
and 
```glsl
	    gl_Position = vec4(0.0,1.0,0.0,1.0);
```
Conclusion: The second coordinate, y, ranges from -1 at the top to 1 at the bottom of the screen. 

If you have dealt with coordinate systems in other contexts (for example, at school),  this may differ from what you are used to. This is not a matter of 
"right" or "wrong", but of convention. And these directions are what Vulkan chooses. 

You should be able to guess where the point 

```glsl
	    gl_Position = vec4(0.8,0.4,0.0,1.0);
```
ends up, even before you run the program. We return these first components to 0.0 and play with the third one: 

```glsl
	    gl_Position = vec4(0.0,0.0,1.0,1.0);
```
Looks the same as before.
```glsl
	    gl_Position = vec4(0.0,0.0,-1.0,1.0);
```
Oh. Now it's gone. 
```glsl
	    gl_Position = vec4(0.0,0.0,0.0,1.0);
```
Still exists in the middle of the window. (We've had this point before.) 

We conclude: The third component, z, does not run between -1 and 1. Actually, it runs between 0.0 and 1.0 (convince yourself that points with z=-0.01
or z=1.01 are not visible) — and does not seem to have an effect. 

There *is* a third coordinate. It is pointing from the screen (at z=0) to a point *behind* the screen (at 1.0). Imagine your screen to be a box. The
screen together with some part behind as the box [-1,1]x[-1,1]x[0,1]  (that is, x between -1 and 1, y between -1 and 1, and z between 0 and 1). Why
doesn't z matter? Well, your screen probably is not a box, but a (mostly) flat surface. If something is in the box "behind" the screen at coordinates 
(x,y,z), there is no better option than to show it on the screen at the corresponding point in the x-y-plane, which means at (x,y,0). 

Some projection like this is necessary whenever you want to turn three-dimensional objects into something that fits onto a two-dimensional surface
(like your screen, that is to say: basically always). Nevertheless, for Vulkan (and before this very final step), we (mostly) can pretend that your
screen actually is a box and every point has coordinates with three components. 

By the way, this coordinate system is "right-handed": Take your right hand, extend your thumb, extend your index finger. (With some imagination, they
should form a right angle between them.) Then take your middle finger and extend it halfways, such that it points in a direction perpendicular to both
thumb and index finger at the same time. We will pretend that the thumb indicates the x-direction (its base being 0 and its tip pointing towards 1),
the index finger indicates the direction of y (again: base at 0, tip pointing toward 1) and the middle finger is z (rest of the hand at 0, tip
pointing toward 1). 

You can now turn your hand such that these finger-axes coincide with Vulkan's coordinate axes ("right"/"down"/"into the screen" — admittedly, you may
have to stand up for this to be less uncomfortable, it's not the easiest rotation of your hand if you want to do it in front of your screen).
The point is, you can similarly form a coordinate system with your left hand (thumb: x, index finger: y, middle-finger (halfway extended): z). No
matter how you turn it, this coordinate system will never coincide with Vulkan's: One finger will point in the opposite direction than it should.  

Now let us have a look at that mysterious fourth coordinate: 

Let's take 
```glsl
	    gl_Position = vec4(0.8,0.4,0.0,1.0);
```
and compare it to 
```glsl
	    gl_Position = vec4(0.8,0.4,0.0,2.0);
```
Hm. That seems to have an effect that's not immediately obvious. 

But isn't this the point halfway between the origin (0.0,0.0,0.0) and the previous point? 

Let's see: 
```glsl
	    gl_Position = vec4(0.4,0.2,0.0,1.0);
```
Yes. That seems to be the same place. Indeed, (x,y,z,w) and (x/w,y/w,z/w,1) are the same point. 

Why would we want to do that? 

It's time for a short excursion about how we represent points and how we transform them. After all, it would seem very reasonable to encode points as
"vectors" with the three components x,y,z and use standard "linear algebra" tools to work with them. At the end of that excursion we will notice one
or two shortcomings that can be addressed by introducing this fourth component. 

[Math lecture](014_Vectors_Matrices.md) coming up ... 
