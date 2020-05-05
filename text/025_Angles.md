## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Angles, orthogonality, rotations

Last chapter we have used some rotations to manipulate positions; this will not have been the last time for us to dig into coordinate changes and
vector maths, so let us have a look at a few more 3D math basics. For some cases we will continue to use the functions provided by nalgebra; but that
should not keep us from looking at the background.

If you are already familiar with the length of vectors, angles, sine and cosine, scalar product (dot product), orthogonality, projection (as in: projection of
one vector onto another), orthogonalization (including Gram-Schmidt process and cross product) and rotation matrices, you can safely skip this chapter; we won't change the code of our program. 

Although we have extended the vectors to use for 3D graphics to vectors with four components, let us stick with those with three (or, occasionally,
two) components. (For their relation with those four-component vectors see [Chapter 15](015_Not_so_linear.md) and [Chapter
16](016_Homogeneous_Coordinates.md).)

Most of these concepts are needed for operations before we have to think about "how to get it on the screen".

### The length of a vector 
Everyone has an idea what a length *is*, we only have to find a way to measure it. 

For vectors like 
<img src="svg/025_-4df87f87b1453c6c8b5c2e8b530973155023d6acadf1e57b8d76fd0642cdab2b.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.2121pt" height="67.09143pt"/> or 
<img src="svg/025_-2accd3842f305bf22aad0431cc621cb25e99f2f892345456698b93617900f47d.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.2121pt" height="67.09143pt"/>, that is easy: The length 
<img src="svg/025_-b2dc70155805524820382b28ce6a7e3a9eb947cc7c0f6c97246ac35942c85aad.svg?sanitize=true?invert_in_darkmode" align="middle" width="28.666714pt" height="24.545448pt"/> is just the value of the non-zero component:

<img src="svg/025_-44791a6e32c7ce1c2f8f3cb3f668377ad15e1e07919a371344e0196f016870ae.svg?sanitize=true?invert_in_darkmode" align="middle" width="58.66662pt" height="24.545448pt"/>, 
<img src="svg/025_-2c8f69b8c161029f016cf74a9dd311cfda5e2ee1c9deb38215fae8b362268352.svg?sanitize=true?invert_in_darkmode" align="middle" width="58.66662pt" height="24.545448pt"/>. 



<p align="center"><img src="svg/025_-ba038fadc315485a18565ea3cd1f805e594a67c8cdc78223fb5e3ab127cd1270.svg?sanitize=true?invert_in_darkmode" align="middle" width="86.39395pt" height="58.909664pt"/></p>

eeeehm, okay, the *absolute* value (i.e. flipping signs whenever it would be negative) of the non-zero component: 
<img src="svg/025_-ee4fb717f7a65ab9ef496d26b57bad3b9376c9a1bb1376fab6988c016c9ad1c3.svg?sanitize=true?invert_in_darkmode" align="middle" width="117.757355pt" height="24.545448pt"/>.

When a vector has two non-zero components, like 
<img src="svg/025_-5fd6368628ac741ec86ff7cf5ca8980031ef5b5f7cfe64adc019bfc4d2bfe539.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.2121pt" height="67.09143pt"/>, the situation looks like this: 
 

<p align="center"><img src="svg/025_-ef2314ccbb3cc19b3b1ef86384292b6117b6d9d94e2218206bae65614ee4127a.svg?sanitize=true?invert_in_darkmode" align="middle" width="103.799126pt" height="101.36162pt"/></p>

and identifying a right triangle, we use Pythagoras' theorem: the square of the length of the green line is the sum of the squared lengths of the blue
lines: 
<img src="svg/025_-bf3364eaf36924369c92bc2292f3713a0519e48a97a55e8bc6f9e37c3276378f.svg?sanitize=true?invert_in_darkmode" align="middle" width="107.39659pt" height="27.28537pt"/> or 
<img src="svg/025_-3582c07084ab3b273e207fe2cc5a3f1c4665ecbf8e4ff1a7f560ca499f49c6f3.svg?sanitize=true?invert_in_darkmode" align="middle" width="140.1361pt" height="28.904388pt"/>. 

How about an actually three-dimensional setting? Say, 
<img src="svg/025_-288a3ad5e2f78bc1b8721c163e117339ea4febfd83d016b5439aeb1dece1bede.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.2121pt" height="67.09143pt"/>? 

That looks like this: 
 

<p align="center"><img src="svg/025_-b8b3fc0e6c4b510c0d499b8300ec4e6ebb3d786b1d0203923416411bb931dffa.svg?sanitize=true?invert_in_darkmode" align="middle" width="94.02152pt" height="133.93507pt"/></p>

and there is, again, a right triangle. One of its sides is just the length we have computed previously. 
<img src="svg/025_-e6680e6e8be34b39c9a4bc090ef0d76621266c4075d262146e39b17d3c279099.svg?sanitize=true?invert_in_darkmode" align="middle" width="230.99483pt" height="27.28537pt"/> and

<img src="svg/025_-410a9f54a8cc6fb97a5792892ab0523dcc5f44344db013c35a28894c1e6439fc.svg?sanitize=true?invert_in_darkmode" align="middle" width="253.5981pt" height="28.904388pt"/>. 

In the same way, we can decompose every vector in right triangles and, as general formula for the length of 
<img src="svg/025_-6e411caf1456628e381ca94537434bde8603859c2e8aa60741f2452771593a9d.svg?sanitize=true?invert_in_darkmode" align="middle" width="72.87116pt" height="67.09143pt"/> obtain 


<p align="center"><img src="svg/025_-b92688842bb890a732bf741dd5c00025b6b894faa66fd3c9f7b8e4f0f3073027.svg?sanitize=true?invert_in_darkmode" align="middle" width="147.92401pt" height="20.069336pt"/></p>


For any 
<img src="svg/025_-e15b4c244ea0d1e5e452fc73457ea30deb78673e2a98501f9038c0626ee67f32.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.909035pt" height="22.72728pt"/>, we see that 
<img src="svg/025_-8bd029529bba4f7bc0af94808604ebc425f8bf0d4b0ce6de46fd7fdfe759e045.svg?sanitize=true?invert_in_darkmode" align="middle" width="89.7652pt" height="24.545448pt"/>. The recipe for normalizing a vector (and thereby obtaining a "unit vector") thus is simple: Instead of 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> use 
<img src="svg/025_-f954167b8ff222bbcc1347b4083ef2f16cb950db6527257d22ed4a29985f3e5f.svg?sanitize=true?invert_in_darkmode" align="middle" width="30.3221pt" height="28.294647pt"/>. It still points in
the same direction, but has length 
<img src="svg/025_-6bd6333c6d55edd0598f05f022c4ee1027b8d0a6afb5d01cd1b2b15022b05933.svg?sanitize=true?invert_in_darkmode" align="middle" width="126.09854pt" height="28.294647pt"/>. This (not only this formula, but normalizing in general) does not work for 
<img src="svg/025_-7e53bb4fa032bd0a9427a81703302a8bc41dd431870f0ecedf1a5384f43b370b.svg?sanitize=true?invert_in_darkmode" align="middle" width="43.064312pt" height="21.090912pt"/>.

### Angles between vectors

Next, we consider the angle between two vectors. For any pair of vectors there is a plane containing both, so we can restrict all illustrations to 2D. 

Angles — the "what lies between" of two vectors. How to measure the "how much"?

First observation: The following angles are the same: 

   

<p align="center"><img src="svg/025_-aff76cc30f3eba14d77d7c7c277e9bf789b9f9318ed400f12ed70f03600cd37f.svg?sanitize=true?invert_in_darkmode" align="middle" width="161.95604pt" height="124.21255pt"/></p>

    

<p align="center"><img src="svg/025_-63f8d74bed20e879f6ca4ca98206128255d658f57e61e6bd442101a4df89fd9a.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="124.21255pt"/></p>


We only have to think about angles between unit vectors. 

Also, rotation (of both vectors by the same amount) does not change the angle, we can try to measure the angle between 
<img src="svg/025_-fc52642d0576571514b0c0bde5e9bbbc701c8c36cc33946b8cf9563f138d5616.svg?sanitize=true?invert_in_darkmode" align="middle" width="39.54551pt" height="47.454872pt"/> and the second
vector: This is still the same angle as before: 
    

<p align="center"><img src="svg/025_-e9706617e84fb5f7db36c2e230a0dc306b4e5ea9b73f79b97b9ccc2bc3081b4b.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="124.21255pt"/></p>


Now, how large is it? As measure for an angle, we use the length of the curve from (the tip of) one to the other, along the circle. (Remember: we are
talking about vectors with length one, so both tips lie on the circle with radius one, if we have the vectors start at the origin.)
    

<p align="center"><img src="svg/025_-ab0268d9645186f96e2938cf189813d01305632591f299df9b204e41ba3cabbf.svg?sanitize=true?invert_in_darkmode" align="middle" width="196.3133pt" height="196.75438pt"/></p>


The full circle therefore corresponds to an angle of 2π. (And if we define ° ("degree") to be the number π/180, then you can call this full circle
angle 360° if you prefer.) 

     

<p align="center"><img src="svg/025_-25b6a36e5260ea9fbf58e577d1d83b26eeec0094f9891040818cf70a6438791d.svg?sanitize=true?invert_in_darkmode" align="middle" width="197.21017pt" height="197.65105pt"/></p>


What about an angle of 9/2 \* π? Well: 

   

<p align="center"><img src="svg/025_-9d4d5507c63ae35f1628c741319874874e9aaf381f4a468c3f877587f86b5c61.svg?sanitize=true?invert_in_darkmode" align="middle" width="143.62686pt" height="149.15286pt"/></p>


No discernible difference to π/2. 

What about negative angles? Well, okay, let's say that is just "in the other direction": 
     

<p align="center"><img src="svg/025_-e11d6e13d39b27b80959888afc6fc79493191e42ee94c1d9d48c3adfe365838c.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="172.74278pt"/></p>

(this, for example, would be an angle of -π/4 from the x-axis to the other vector) 

Introducing a sign, a direction means that we are not so much talking about an angle *between*, but rather about an angle *from* ... *to* ...

How do we know which of the following two angles is +π/4 and which is -π/4? 

     

<p align="center"><img src="svg/025_-bd1e5907faeb973b5c9c29643460bf8683370c09a1e8af57583fd9fb34a748ec.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="172.74278pt"/></p>

That is a matter of convention. We decide: The angle from 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> to 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/>is *positive*. 

So, this angle is +π/4: 



       

<p align="center"><img src="svg/025_-215458e275ff0ff4f54e48940779a08257d576ea2baedce2a17430d27affd7cc.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="124.21255pt"/></p>



and this angle is +π/4

      

<p align="center"><img src="svg/025_-cb1afd1d97aeb4f9e4705cbc1177831671848ef8b101935486e55493fba977fb.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="218.01227pt"/></p>


Aren't they the same? No. Very important, but very subtle difference: Pay attention to the direction of the axes! (And, please, never draw an axis
with an arrow head on each end. The arrow does not mean "there is more in this direction", it means "the numbers increase in this direction".)

For some of my drawings, the directions do not correspond to those of [Vulkan's coordinate system](013_Coordinates.md). I don't care: That's a matter
of "from which direction are we looking at it". But it becomes the more important to look at the directions indicated at the axes.

This angle is, indeed, -π/4 (as mentioned before, but now you can appreciate the direction of the axes...):
      

<p align="center"><img src="svg/025_-509e9ae145cefe8a3f7f3db4c2ecad8d0d7cbb04504a1334292f75961b08f612.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="172.74278pt"/></p>


This is a nice definition, which works well for angles in the x-y-plane -- but doesn't really help outside of it, and we are hoping for 3D in the end. How do we distinguish positive and negative angles there? 

Two ways of dealing with it (both will occur): 

1. We don't. In some cases we really care about an *angle between* and not an *angle from to*, in that case the sign doesn't matter, so we don't
   bother defining it.
2. We specify around which axis we are turning (think of the angle as rotating the first vector on top of the second) and define to which direction of
   turning positive rotations correspond. For our rotations "in the x-y-plane", we will consider them "rotations around 
<img src="svg/025_-02078dbc2d8cb8dc7a281535c130bb37e051a0c77a85c429e16acf67f1740095.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/>".  In which
direction? What does rotating around a vector mean? Take your right hand. Extend the thumb. This is the vector around which to rotate. In which
direction? The remaining (non-extended) fingers with a little bit of imagination form an arc indicating a screw direction. (If your thumb points
towards your nose, you will perceive this as counterclockwise.) That is the positive direction. Check that rotating 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> around 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> by +π/2 actually results in 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/>. 

Finally, a symbol, in case I don't want to write "angle between 
<img src="svg/025_-7d0cc33630acdb721ed01d26442b8cefa1cd2f5324c2213dfc879935c81c959b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/> and 
<img src="svg/025_-016397ad676b29c5abad0b1fd24cfc6e0ca56652405fdbd7f93b27ab66fbc3d5.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/>" in formulae: 
<img src="svg/025_-e271914a39ea6cc7ee68212ed012af8d100bde4f7f03c91fc83aed159234ae87.svg?sanitize=true?invert_in_darkmode" align="middle" width="66.42432pt" height="24.545448pt"/>.


### Sine and Cosine 

The above was: Given two vectors, what is the angle. Now we look at the opposite: Given an angle (say, φ), can we find two vectors with that angle between
them? Let's, again, fix the first as 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>. 

       

<p align="center"><img src="svg/025_-194c715954d8c326d86787814885c83a69c10a7c5ea3ee7b5a63ab9b46c39c15.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="124.21255pt"/></p>


Since we know the angle, that is: the arc length shown in the picture, the vector is easy to find graphically: 

 

<p align="center"><img src="svg/025_-11a735a87c238c1412dfb86ce97734e4eb9c9f3f207f5da84ae62d4f949937cf.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.33963pt" height="124.21255pt"/></p>


What are its coordinates? Let's cheat. They are - apparently - uniquely defined by the above procedure (draw part of the unit circle, starting at
(1,0), arc length equal to φ, and take the coordinates of the endpoint). We can just give names to these results "the x-coordinate of this point"
(txcotp), "the y-coordinate of this point" (tycotp) and refer to txcotp(φ) and tycotp(φ) in all computations where we need them. If they are useful enough, many people will use them, and we
will finally get functions in Rust and GLSL that immediately give us these values if we hand them the value of φ. 

Well, the names that have traditionally been used are different, but the idea is not too far off: txcotp is called cos and tycotp is sin. The point at
the tip of a unit vector starting from the origin that has an angle φ to the x-axis, therefore is 


<p align="center"><img src="svg/025_-c83532add76f2abdc9731ee2c66fa22e4e8e61f1638e1357fd69c29adeb5103d.svg?sanitize=true?invert_in_darkmode" align="middle" width="66.70453pt" height="39.273117pt"/></p>

(And whether we write cos φ or cos(φ) is merely a question of taste and readability.)



<p align="center"><img src="svg/025_-aec2688d3f0fa69b028413424af1688945f00a6fbab6380e72c4df7382e40b72.svg?sanitize=true?invert_in_darkmode" align="middle" width="196.3133pt" height="196.75438pt"/></p>


By the way: As we have defined sine and cosine by means of a point on the unit circle, essentially *by definition* we have that 
<img src="svg/025_-4fc4d8422d52f033efa8e993568fe973760905d0cf524a02e54354c99eaab62d.svg?sanitize=true?invert_in_darkmode" align="middle" width="183.06062pt" height="27.28537pt"/>. (Again a note on writing: This is often shortened to 
<img src="svg/025_-019a6d918301b0c72063096871adb187b4b339e91c8fd50c9f88d4d05871095d.svg?sanitize=true?invert_in_darkmode" align="middle" width="140.33302pt" height="27.801567pt"/>, although 
<img src="svg/025_-351127bba29bfedc803c65babdcf3b262becd35e0895807ea1cfb8c092752f22.svg?sanitize=true?invert_in_darkmode" align="middle" width="49.712036pt" height="27.28537pt"/> could be mistaken for 
<img src="svg/025_-9361ed804e6425c039b3a48183bfb9686cc118fb72be6f10eecdd9161fd6df0e.svg?sanitize=true?invert_in_darkmode" align="middle" width="87.25008pt" height="24.545448pt"/> —
but the latter is so uncommon that that is no real concern.)

If we already know the cosine, but want to know the corresponding angle: The answer to that question is called arccos, so that 
<img src="svg/025_-e9b324b665c3ebbc1406446f07d54240a749f2e53ea46919648aa8366be80ee6.svg?sanitize=true?invert_in_darkmode" align="middle" width="131.63625pt" height="24.545448pt"/> —
at least for angles φ between 0 and π. If we admit larger angles, there are different possibilities that result in the same cosine.

Similarly, arcsin is the answer to the question "which angle has the given value as its sine?"

And while we're at introducing trigonometric functions: The tangent is the quotient of sine and cosine: 
<img src="svg/025_-fd4513427ae86c315655865a58075b7e63c8045dfd62dfce7970f2e1b2b5d6bf.svg?sanitize=true?invert_in_darkmode" align="middle" width="100.80855pt" height="30.82972pt"/>.

### Rotations in 2D

Now that we have names for the coordinates of a point on the x-axis after rotating it by a certain angle around 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/>, let's try to describe the
rotation of an arbitrary point (in the plane) by a given angle φ.

Rotations are linear (think about it; if necessary, have another look at the introduction of linearity in [Chapter 14](014_Vectors_Matrices.md)), thus
can be represented by matrices. 

Say, we want to rotate by an angle φ, and call this rotation 
<img src="svg/025_-b24866d31507f3994428e3240960a498fc41f234075055407a675784ff46e76e.svg?sanitize=true?invert_in_darkmode" align="middle" width="25.247616pt" height="22.36361pt"/>. Then 


<p align="center"><img src="svg/025_-75f0d36d607de44d1f9fb9ad079693b194f533ce1777f3058882281c7b20cb97.svg?sanitize=true?invert_in_darkmode" align="middle" width="105.02964pt" height="39.273117pt"/></p>

How to find the values of "?"? The first column was the outcome of applying the rotation to 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>. Well, that's not too difficult, we have just given
names to those components: 


<p align="center"><img src="svg/025_-f7c07fa1f76af1837d467eefe4be33b01b1666fae2eca4088c229786a0c9a034.svg?sanitize=true?invert_in_darkmode" align="middle" width="132.6432pt" height="39.273117pt"/></p>

The second column is what happens to 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/>. And 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> is rotated to 
<img src="svg/025_-fefdc3860443bb48ffae2a184f5b1ff1bcb019e86ae614ff7ea1f62f72197099.svg?sanitize=true?invert_in_darkmode" align="middle" width="80.340904pt" height="47.454872pt"/>.


If that was too fast or you prefer pictures: 



<p align="center"><img src="svg/025_-4db9aca4c2f814c97450d61d3d368297562a05a1cd7ed29188f161dd692a9fde.svg?sanitize=true?invert_in_darkmode" align="middle" width="254.79845pt" height="124.21255pt"/></p>


The final matrix hence ends up being 


<p align="center"><img src="svg/025_-3db54b1a774d4654e5b2e2ca01c3d4fbea398f4d1866c8012fe6a21bac58400e.svg?sanitize=true?invert_in_darkmode" align="middle" width="173.89313pt" height="39.273117pt"/></p>


What happens if we rotate first by an angle 
<img src="svg/025_-0bf3a7cb8da4af306f9fb09474c1cc6ff538b47483c3f8d4a5a40c788d3650b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="21.601288pt" height="14.090904pt"/> and then by an angle 
<img src="svg/025_-65e72b597c73202256eb0de3dcc341eca4a387cca786156c0866434901b79cee.svg?sanitize=true?invert_in_darkmode" align="middle" width="21.601288pt" height="14.090904pt"/>? Well, in total, we rotate by 
<img src="svg/025_-e804a5d8419b55583b8f10e39da4ee22e30bcf9223bd97c2c7d8021fbab4df72.svg?sanitize=true?invert_in_darkmode" align="middle" width="59.40422pt" height="19.09092pt"/>. Can we see that for matrices?
Yes: 
<img src="svg/025_-151bb5326b5af87ddde01643c4f73745fa51ac8033893e8f851a3af4b23f7c53.svg?sanitize=true?invert_in_darkmode" align="middle" width="132.28064pt" height="22.36361pt"/>. 

Oh, we have never before covered how to multiply two matrices. Then let's have a look now. What is 


<p align="center"><img src="svg/025_-83b38597e9205751b2f4009e0bac9042fa4e25138fb9ab1945ef20000b4fb1ca.svg?sanitize=true?invert_in_darkmode" align="middle" width="123.855896pt" height="39.273117pt"/></p>

We know how to multiply matrices with vectors. Let's see:


<p align="center"><img src="svg/025_-ef0ffe31f8a39761256c74fc0561fa9f3d30294cc0eb0aa407fd9b03e0faef67.svg?sanitize=true?invert_in_darkmode" align="middle" width="1138.9797pt" height="39.273117pt"/></p>

so: 


<p align="center"><img src="svg/025_-aa938dd6555e0c86e0268f61053b00a43eb827443a2e49a14b7f3a3e3d51371d.svg?sanitize=true?invert_in_darkmode" align="middle" width="290.81097pt" height="39.273117pt"/></p>

In position (1,2) (that is, first row, second column) of the result, we get the product of the first row of the first matrix and second column of the
second. (And product of first row and second column is something we can interpret in terms of "matrix times vector": 
<img src="svg/025_-af326070ca91e9ffc7a0d6a39137850d94f9e965eec9330f3c0ce941fbd3656c.svg?sanitize=true?invert_in_darkmode" align="middle" width="166.17122pt" height="47.454872pt"/>.) 

Okay, with that out of the way, back to our rotations: 


<p align="center"><img src="svg/025_-8faf952bccc70a189abf9aca9fcd17d8c3bfa70116354cf8151bb19b4a4ed9c5.svg?sanitize=true?invert_in_darkmode" align="middle" width="1143.307pt" height="39.273117pt"/></p>


We learn (by comparing components of the first and the last matrix in that short computation): 


<p align="center"><img src="svg/025_-9f44ff3ad437c836a1b96c6257ecbee04aeb8842bc3e3f5aa4c907e8dcb47241.svg?sanitize=true?invert_in_darkmode" align="middle" width="302.88864pt" height="16.363632pt"/></p>

and 


<p align="center"><img src="svg/025_-02adc81bb6caed7401e34239a95dcba9e9872865beaf5fff7bad25b1ed929102.svg?sanitize=true?invert_in_darkmode" align="middle" width="301.0705pt" height="16.363632pt"/></p>

(These are very useful identities.)

### *Computing* the angle between two vectors
If we are given one vector v and want to compute the angle between 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> and this vector, that is not too difficult: We normalize v (all of the above
was for unit vectors), and then take its first component: That is the cosine of the angle. 

But what if the other vector is not 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, if we have two (unit) vectors 
<img src="svg/025_-7d0cc33630acdb721ed01d26442b8cefa1cd2f5324c2213dfc879935c81c959b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/> and 
<img src="svg/025_-016397ad676b29c5abad0b1fd24cfc6e0ca56652405fdbd7f93b27ab66fbc3d5.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/>? 

Well, they are unit vectors, so let us write them as 
<img src="svg/025_-0e76db1564f172ddaeb4d29905fd0ba778283d50b1a169418ebbe314151f7222.svg?sanitize=true?invert_in_darkmode" align="middle" width="107.92417pt" height="47.454872pt"/>, 
<img src="svg/025_-04f918373f5eafa05c16a00d2c091c7b331dbd010f51d90393d602e04e9b652a.svg?sanitize=true?invert_in_darkmode" align="middle" width="107.92417pt" height="47.454872pt"/>. And an angle should not change when
we rotate both vectors, so the angle between 
<img src="svg/025_-b745a1e53993f4fc537a651918164c1099f14a6aca2a01a659430c3b2a937c98.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.379738pt" height="22.36361pt"/> and 
<img src="svg/025_-831d9ca2c4b365c2e782753438a9f786274af27db53162e49a7c1e325881b0fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.379738pt" height="22.36361pt"/> is the same as the angle we want to find, for every rotation R. Let's pick the rotation
by 
<img src="svg/025_-0f1fb283d672720c1bd9d67168b0c47c282d402294548d8fc86b8d7f70a2223b.svg?sanitize=true?invert_in_darkmode" align="middle" width="41.601196pt" height="19.09092pt"/>, which turns 
<img src="svg/025_-7d0cc33630acdb721ed01d26442b8cefa1cd2f5324c2213dfc879935c81c959b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/> to 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>. Because then we are back in the previous setting. 

So: 
<img src="svg/025_-9ee74a3d5e83696905b05e30e3d8e47cc25c6012dae55d027eeca7f1b7135ad5.svg?sanitize=true?invert_in_darkmode" align="middle" width="402.37103pt" height="47.454872pt"/>. (If the changes of signs in the last step
confuse you, try to follow them along the definition of sine and cosine and negative angles in the defining "picture".)

What are 
<img src="svg/025_-b745a1e53993f4fc537a651918164c1099f14a6aca2a01a659430c3b2a937c98.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.379738pt" height="22.36361pt"/> and 
<img src="svg/025_-831d9ca2c4b365c2e782753438a9f786274af27db53162e49a7c1e325881b0fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.379738pt" height="22.36361pt"/>? Let's compute: 


<p align="center"><img src="svg/025_-2889175f4787ab8c5f2d6c4b6b8a08f1330fa82796aff9a65b934d328416175d.svg?sanitize=true?invert_in_darkmode" align="middle" width="583.237pt" height="39.634537pt"/></p>

Okay, that's what we wanted, because now we only need the angle between 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> and the (rotated) second vector.


<p align="center"><img src="svg/025_-3d8010048473f109df50b6d0ef1afc79c436d8890e959ed5aaf0777fb32d15e8.svg?sanitize=true?invert_in_darkmode" align="middle" width="529.1461pt" height="39.273117pt"/></p>

And the cosine of the angle is the first component: 


<p align="center"><img src="svg/025_-593290aa1701060a6db75bd598e9560e76c1846092219adc8d26e319ba303481.svg?sanitize=true?invert_in_darkmode" align="middle" width="190.82822pt" height="14.11042pt"/></p>

(We can also obtain this term by computing 
<img src="svg/025_-e3501891e19a8c6b8c80ddcbb977bb54694ea64221276ee4441484537c8467b1.svg?sanitize=true?invert_in_darkmode" align="middle" width="95.562485pt" height="24.545448pt"/>, what a surprise.) 

So, from two vectors 
<img src="svg/025_-2a65a53553b4d4f6cf64edabcf40d183e4d8280e9dfeb446f9478ee4a315621e.svg?sanitize=true?invert_in_darkmode" align="middle" width="73.80301pt" height="47.454872pt"/>, 
<img src="svg/025_-4a0572bceaf6c1f063cdc02ffa8136ed074d62f1e452e29a1ad70676fe5ed551.svg?sanitize=true?invert_in_darkmode" align="middle" width="73.80301pt" height="47.454872pt"/> we ended up with 
<img src="svg/025_-4f5d0c1887eb5204ea009e5259ca4b7110c93887631a3c89d5d48c148becd661.svg?sanitize=true?invert_in_darkmode" align="middle" width="198.10089pt" height="21.857208pt"/>. More general: 


<p align="center"><img src="svg/025_-1d92c37b1241494ac30726e1853e47d116599476171ab4741ff33e024b11c8d8.svg?sanitize=true?invert_in_darkmode" align="middle" width="153.45447pt" height="39.273117pt"/></p>

That's — easy to compute. And useful in connection with angles. We should give it a name: Scalar product. Inner product. Dot product. 

Formula to remember:


<p align="center"><img src="svg/025_-69c5bf77e155b194c560fcf2d87dd588b9e114712a1345010b866ec4d33dcf06.svg?sanitize=true?invert_in_darkmode" align="middle" width="201.18167pt" height="16.363632pt"/></p>

(I have included the lengths of the vectors in order to account for the fact that we were using unit vectors all the time.) 

The nice thing: This formula still works in 3D. Here, 
<img src="svg/025_-5266236f632399265bda20c05ab488a3cf46cf3884bb041ef3e2bfe323c1e38c.svg?sanitize=true?invert_in_darkmode" align="middle" width="262.8965pt" height="67.09143pt"/> (as you may have expected),
and the "formula to remember" is the same.

### Orthogonality
One of the most important sizes an angle can have is π/2; the right angle. The angle between two vectors is a right angle (read: "the vectors are
orthogonal to each other") if its cosine is 0, that is to say: If their scalar product is zero. Very good, because very simple to check. 

Finding out whether 
<img src="svg/025_-671ff9e85dd65eb1213ec2ca8840475c108808509c5f3a99446fe8ffbe73087e.svg?sanitize=true?invert_in_darkmode" align="middle" width="190.72704pt" height="47.454872pt"/> does not even involve sine and cosine. 

And also in 3D: 
<img src="svg/025_-474572c8a28d02a12d2aed025b0ff02f8ea358e70df002441bdcc076851fa687.svg?sanitize=true?invert_in_darkmode" align="middle" width="86.481pt" height="67.09143pt"/> and  
<img src="svg/025_-d943900173e9a872e0f068f77e83a0a31f92c8f423b7d7d519ae08437eb358ce.svg?sanitize=true?invert_in_darkmode" align="middle" width="86.481pt" height="67.09143pt"/> are orthogonal iff 
<img src="svg/025_-82a694260d3d4463a20e58ad63bbfa79afeb78e85074cfd4d07dd6ea7f63ad23.svg?sanitize=true?invert_in_darkmode" align="middle" width="167.10597pt" height="21.090912pt"/>. 

Suppose we are given two vectors (
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> and 
<img src="svg/025_-8adfc51cd827f19242518af264c7d62d3331d37727d899a125f940d47957e0a8.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.164806pt" height="14.090904pt"/>), and we want to decompose one of them (say, 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>) in two parts: One part with the same direction as the other given vector (let's call this part 
<img src="svg/025_-5c55c60300311e59f01473c7f048328ec567b680449aab8152c373b4626efda9.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/>), and a second part orthogonal to that (
<img src="svg/025_-107f9d95c61db132d5a8e78f80a76d0077d7eb5e26d2ef0bf7ff91ff8f868977.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.357067pt" height="14.090904pt"/>). 



<p align="center"><img src="svg/025_-388beea1f40a8596dd15dc6dd1a22b45ee3969464570c183de928e7328a0215f.svg?sanitize=true?invert_in_darkmode" align="middle" width="111.134094pt" height="77.76769pt"/></p>




<p align="center"><img src="svg/025_-6be539a7e3b98dec31c9b32a975d762214039326e74042891c6c0ad43cdcc331.svg?sanitize=true?invert_in_darkmode" align="middle" width="111.13408pt" height="79.09789pt"/></p>


If 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> had a length of 1, then 
<img src="svg/025_-5c55c60300311e59f01473c7f048328ec567b680449aab8152c373b4626efda9.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/> would have a length of cosφ. Also 
<img src="svg/025_-5c55c60300311e59f01473c7f048328ec567b680449aab8152c373b4626efda9.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.828583pt" height="14.090904pt"/> points in the same direction as 
<img src="svg/025_-8adfc51cd827f19242518af264c7d62d3331d37727d899a125f940d47957e0a8.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.164806pt" height="14.090904pt"/>. 

Apparently, 


<p align="center"><img src="svg/025_-c735a6b2209a4f743c53a961f676d8f89bceb06699585c130dee67344feb346a.svg?sanitize=true?invert_in_darkmode" align="middle" width="120.4425pt" height="36.931103pt"/></p>

 
And we can compute cosφ much more easily, as dot product (between *normalized* vectors e and v):


<p align="center"><img src="svg/025_-3699d7306fd3d1368bcce7c89019fc59b779ed1e1060ad64bc8f87e16babd107.svg?sanitize=true?invert_in_darkmode" align="middle" width="188.53403pt" height="36.931103pt"/></p>

 
(If 
<img src="svg/025_-8adfc51cd827f19242518af264c7d62d3331d37727d899a125f940d47957e0a8.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.164806pt" height="14.090904pt"/> has length one, 
<img src="svg/025_-53762dfbc2565a277cca51256cb37ad82d44f78fe62b2f57cd6b0b7a38498bb1.svg?sanitize=true?invert_in_darkmode" align="middle" width="89.696846pt" height="24.545448pt"/>, much nicer and shorter.) 

Finally, we can set 
<img src="svg/025_-6af3139f1a8f5defadb6b984568f1eb6f0c9a2ed7a949b27125d0e47386ed5b0.svg?sanitize=true?invert_in_darkmode" align="middle" width="87.7243pt" height="19.09092pt"/>.

Recipe: If you have two vectors (say, in 3D) and want to have two orthogonal vectors that lie in the same plane, take the first of your vectors (call
it e), and take the orthogonal part 
<img src="svg/025_-107f9d95c61db132d5a8e78f80a76d0077d7eb5e26d2ef0bf7ff91ff8f868977.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.357067pt" height="14.090904pt"/> of the second vector v. (Note that 
<img src="svg/025_-107f9d95c61db132d5a8e78f80a76d0077d7eb5e26d2ef0bf7ff91ff8f868977.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.357067pt" height="14.090904pt"/> depends on the choice of 
<img src="svg/025_-8adfc51cd827f19242518af264c7d62d3331d37727d899a125f940d47957e0a8.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.164806pt" height="14.090904pt"/>, even though the notation
does not announce this dependency.) Often, you'll want to normalize them.


If we have two vectors 
<img src="svg/025_-41e7e5d87ee76bf8b2cc987630710a0b4813f96dfa3620470090e2b4fc086e1b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> and 
<img src="svg/025_-c91f694db3a1a7510557cc62d9a259e7ff7834ae9fc35b737092263609e1cb5a.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> and want to find a third one orthogonal to both, we can first ensure that 
<img src="svg/025_-c91f694db3a1a7510557cc62d9a259e7ff7834ae9fc35b737092263609e1cb5a.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> is orthogonal to 
<img src="svg/025_-41e7e5d87ee76bf8b2cc987630710a0b4813f96dfa3620470090e2b4fc086e1b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> (by
replacing it by 
<img src="svg/025_-9149324b01055165a284956bbce18b1e493debeb87a40c18f7f8b9da9e4e2c7c.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.924328pt" height="14.090904pt"/>); then we can choose a random vector 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> (if it lies in the plane of 
<img src="svg/025_-41e7e5d87ee76bf8b2cc987630710a0b4813f96dfa3620470090e2b4fc086e1b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> and 
<img src="svg/025_-c91f694db3a1a7510557cc62d9a259e7ff7834ae9fc35b737092263609e1cb5a.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/>, we will have problems. But that's improbable, and we could just start 
with a different vector), and compute 
<img src="svg/025_-a08984eb423fd5878cda939e9f052cdada1f53ce53de0b1a733e2b2409e22f90.svg?sanitize=true?invert_in_darkmode" align="middle" width="56.88669pt" height="25.160019pt"/> with respect to 
<img src="svg/025_-41e7e5d87ee76bf8b2cc987630710a0b4813f96dfa3620470090e2b4fc086e1b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/>. We then take 
<img src="svg/025_-6409acbfa7fc2936a4aa0b4a300e74c8041df0b4de4c8aad7d077ce0aa691c4a.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.509823pt" height="25.160019pt"/> and compute 
<img src="svg/025_-2dbfdd68443e7b94c110fcd17d4a7ef86fd6264a0772697cf03fa20527a94f35.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.357067pt" height="25.160019pt"/> with respect to 
<img src="svg/025_-c91f694db3a1a7510557cc62d9a259e7ff7834ae9fc35b737092263609e1cb5a.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/>. The final 
<img src="svg/025_-2dbfdd68443e7b94c110fcd17d4a7ef86fd6264a0772697cf03fa20527a94f35.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.357067pt" height="25.160019pt"/> is orthogonal to 
<img src="svg/025_-41e7e5d87ee76bf8b2cc987630710a0b4813f96dfa3620470090e2b4fc086e1b.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/> and 
<img src="svg/025_-c91f694db3a1a7510557cc62d9a259e7ff7834ae9fc35b737092263609e1cb5a.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.51608pt" height="14.090904pt"/>. 


But we can also use a shortcut and introduce a ready-made concept and formula yielding the vector that is orthogonal to two given vectors: The
cross-product. 

Actually, in this sentence the "the" in "*the* vector that is orthogonal" is overly optimistic: There are many such vectors; at least they all belong
to one line. If we decide on length and direction (out of the two possible directions on this line) of the vector, we have determined it uniquely.
Given two vectors 
<img src="svg/025_-0f4dfdc4cce7af8edd011919f4d4a651eb7b0366e06de1cd76ba705871adb31f.svg?sanitize=true?invert_in_darkmode" align="middle" width="79.39772pt" height="67.09143pt"/> and 
<img src="svg/025_-41b6f5209010500e5867253d85156e08b9bd88c166e13e2fff75ed799c5a25f2.svg?sanitize=true?invert_in_darkmode" align="middle" width="76.143875pt" height="67.09143pt"/>, we will then call this new orthogonal vector the "cross-product"


<p align="center"><img src="svg/025_-ec90042db276659fbdd82a2a8a893a921264b0933be22fd2dea495c0289ab3ca.svg?sanitize=true?invert_in_darkmode" align="middle" width="114.41475pt" height="58.909664pt"/></p>

Perhaps we revisit it in the future, but for now let me just write the definition: 


<p align="center"><img src="svg/025_-24d70d5b79939a9e44e99329e4723bdc16cf79c6fec0f8fa6e5f2d8b6429f2f8.svg?sanitize=true?invert_in_darkmode" align="middle" width="244.60776pt" height="58.909664pt"/></p>

(From one line to the next, each index is increased by one, where we start again from 1 instead of using 4.) 

This vector is, indeed, orthogonal to 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/>:


<p align="center"><img src="svg/025_-79ddc79483beb2ddfd00fd78d931e0b27c303080c389aa6dc66e1600e138dcec.svg?sanitize=true?invert_in_darkmode" align="middle" width="972.7013pt" height="58.909664pt"/></p>

It is also orthogonal to 
<img src="svg/025_-b81f5324fe79918ee2159f8b11fedb9a1638b475ff3eaea81f7ae5368b777729.svg?sanitize=true?invert_in_darkmode" align="middle" width="11.568189pt" height="22.72728pt"/> (you do the computation). 

Moreover, 
<img src="svg/025_-b5f476c5283446f41a25c006ca7eb5c3fdec628a54cd46702a908c951911af41.svg?sanitize=true?invert_in_darkmode" align="middle" width="70.43552pt" height="22.72728pt"/> (in this order) form a right-handed coordinate system. 

You can either check with the definition or use this last property to see that 
<img src="svg/025_-4bed6f474a37370d9de3f6c72d4ad5fc408d7a0d412db4b6825c28d54128654d.svg?sanitize=true?invert_in_darkmode" align="middle" width="110.43542pt" height="22.72728pt"/>. 


### Rotation matrices (3D) 

How can we recognize a rotation? 

Tying back to the above ideas about angles: If we have some vectors and apply the same rotation to them, the angles between them do not change. Also
their lengths remain the same. 

If we want to put these two ideas into one condition, we can say that the scalar products are unchanged: For every rotation R and every vectors u and
v, we have 
<img src="svg/025_-ed634cd412f2deca3bf4882689eb1d3bd31a4d8fd5b59f82dc3b92b3f6757cb5.svg?sanitize=true?invert_in_darkmode" align="middle" width="136.32947pt" height="24.545448pt"/>. (What does this have to do with lengths? Well, 
<img src="svg/025_-45667156722332a56b26da8acf8b901cc3e7cf491bc4536afc06fcbcff313dcb.svg?sanitize=true?invert_in_darkmode" align="middle" width="82.47347pt" height="27.28537pt"/>.) 

We know that the columns of the matrix representing R are given by 
<img src="svg/025_-6f58e1549d4b94e07d12424aef782be0952394657b228c521f4bd6d7b52e2a11.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.86631pt" height="22.36361pt"/>, 
<img src="svg/025_-207312e8cdc3a14c55b28e2e88eaed0d0aeb9e571412a9998a8099ce7acd67c5.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.423527pt" height="22.36361pt"/> and 
<img src="svg/025_-e3f23e92e30ce0d4527478ffe6ff0cb199be5743a4a67ba522a2fbc79b73b756.svg?sanitize=true?invert_in_darkmode" align="middle" width="31.12949pt" height="22.36361pt"/>. And we know that 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> have length one
and are orthogonal to each other. 

If we know three vectors 
<img src="svg/025_-38f33bbf877ddc852cdb98c53b41501e63942fab97c97ed50c5a1cb79cfb2c6e.svg?sanitize=true?invert_in_darkmode" align="middle" width="80.8334pt" height="67.09143pt"/>, 
<img src="svg/025_-5acdf7c1924a76bee09fbaf1746f752fdce5888ab475e0499b8332d3bcfdb2b8.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.54921pt" height="67.09143pt"/>, 
<img src="svg/025_-04c0a9cfaab7016cf37b4b44cc7bb2a2ac8fbb76b0c15e66f516f732c368f993.svg?sanitize=true?invert_in_darkmode" align="middle" width="85.968765pt" height="67.09143pt"/> that have length one and are orthogonal to each other, we can form a rotation matrix by setting 


<p align="center"><img src="svg/025_-1900d2686b2aa13854941085e1edb6c1572473c283daf01ae2d60ca60f0d9799.svg?sanitize=true?invert_in_darkmode" align="middle" width="150.47598pt" height="58.909664pt"/></p>

(Okay, to be more precise, that's not completely true: We not only get rotations but could also obtain a reflection. But close enough.) 

This is the rotation turning 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> into 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>, 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> and 
<img src="svg/025_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/>, respectively.

How do we find a rotation (say, 
<img src="svg/025_-89bf392c5b21c4672e6ecfa74f684dbb97a5600052fba934205ce8bc2be34b16.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.481813pt" height="22.36361pt"/>) turning 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>, 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> and 
<img src="svg/025_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/> into 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/>? 

Let us give the components of 
<img src="svg/025_-89bf392c5b21c4672e6ecfa74f684dbb97a5600052fba934205ce8bc2be34b16.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.481813pt" height="22.36361pt"/> some names: 


<p align="center"><img src="svg/025_-2c387e8f42fb849f62cb74f996f9d6cdf43cad1ab9655b1653f2f714b02bde5c.svg?sanitize=true?invert_in_darkmode" align="middle" width="147.79579pt" height="58.909664pt"/></p>

This seems strange — until now it was mostly the columns, not the rows of a matrix that we used, because they had some meaning. Why do something else
now? Because in the end it will turn out that this way of giving names is more useful than having columns a, b, c. (You'll see.) 

And, by the way, another random bit of notation (I should introduce it somewhere): If 
<img src="svg/025_-0f4dfdc4cce7af8edd011919f4d4a651eb7b0366e06de1cd76ba705871adb31f.svg?sanitize=true?invert_in_darkmode" align="middle" width="79.39772pt" height="67.09143pt"/>, then 
<img src="svg/025_-a51616fa918b9e2859d5d7be100a693ccb22db1ea2de334d9cbf508bfa5419c7.svg?sanitize=true?invert_in_darkmode" align="middle" width="99.40503pt" height="27.818298pt"/> is called the
transpose of 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/> and denoted by 
<img src="svg/025_-530f382325669510772b4ffc0d2505c8fb8ee162ffc54839ff74a9a6a2f67eea.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.354654pt" height="28.215157pt"/>. So, if you wish: 
<img src="svg/025_-33ae38791299dc3e1d3cba637607115724d5dc870771e9667ecabe87edda7f37.svg?sanitize=true?invert_in_darkmode" align="middle" width="86.49268pt" height="68.915115pt"/>. 

Back to our question: What are 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/>, 
<img src="svg/025_-b81f5324fe79918ee2159f8b11fedb9a1638b475ff3eaea81f7ae5368b777729.svg?sanitize=true?invert_in_darkmode" align="middle" width="11.568189pt" height="22.72728pt"/>, 
<img src="svg/025_-0efacdd4bd126c80c2a74d9efb3aa4eeff0483d97b23902570123d84489ced79.svg?sanitize=true?invert_in_darkmode" align="middle" width="11.626929pt" height="14.090904pt"/> if we want 
<img src="svg/025_-89bf392c5b21c4672e6ecfa74f684dbb97a5600052fba934205ce8bc2be34b16.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.481813pt" height="22.36361pt"/> to rotate 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/> onto 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> etc.? 

Well: 


<p align="center"><img src="svg/025_-5a8b64a2e0fdff0dfd5dedf1bbcd409f51db6bb5488d810358918c0192ed911e.svg?sanitize=true?invert_in_darkmode" align="middle" width="58.891594pt" height="14.363623pt"/></p>

 
That is: 


<p align="center"><img src="svg/025_-62831cdec9ee348b1dcaca0490b0f852848941854eb77881725f413403eef784.svg?sanitize=true?invert_in_darkmode" align="middle" width="292.213pt" height="59.821503pt"/></p>

or, separately: 
<img src="svg/025_-d70ee4b44e029a05b21602ada3c675fa8ad54908a59cbbc50023a2ab9b96e604.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.469254pt" height="28.215157pt"/>, 
<img src="svg/025_-260315199762a6c4d3b150402b257c834581f2b17d30e60c95ea9668c13a2646.svg?sanitize=true?invert_in_darkmode" align="middle" width="60.842323pt" height="28.215157pt"/>, 
<img src="svg/025_-810ab82bb777cccd2583520e661d2c739f7f88963aa486ed2b669ed15e1eef76.svg?sanitize=true?invert_in_darkmode" align="middle" width="60.901062pt" height="28.215157pt"/>. If you prefer scalar products, that is the same as 
<img src="svg/025_-ebf37169094cb244327c5e0ca13c585e9b4287704bf18075964d89c7c91375be.svg?sanitize=true?invert_in_darkmode" align="middle" width="64.38058pt" height="21.090912pt"/>, 
<img src="svg/025_-f141a4ea877a68289c2000c6d4123abdd6b85df9501a2f879cf3caa810e7e5b8.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.753647pt" height="22.72728pt"/>, 
<img src="svg/025_-d5c591c583b56820d02d4e008ceaa7410c50748cd9def1bab4fb03806e86d988.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.812393pt" height="21.090912pt"/>. (Check it
with the definitions.) 

Treating 
<img src="svg/025_-c25c892e46b620a8fe0ffdc61f4de617d5fe874f452f68f4dac64dbea8bc9625.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.14573pt" height="22.36361pt"/> and 
<img src="svg/025_-abd2ce708c4a413bdd163f0de31c708442beae55ba8a2349bf9c138f5b666cf2.svg?sanitize=true?invert_in_darkmode" align="middle" width="65.48807pt" height="22.36361pt"/> in the same manner, we also find 
<img src="svg/025_-245e818af2a38c9064b1f25f3893f8345ffc875137577af01d3ce3cebd790d51.svg?sanitize=true?invert_in_darkmode" align="middle" width="63.53203pt" height="21.090912pt"/>, 
<img src="svg/025_-340e7b6c0ae63d48e69eed008fb591aef7b9e8cfe0c29a3c68976113c46b33a0.svg?sanitize=true?invert_in_darkmode" align="middle" width="61.905098pt" height="22.72728pt"/>, 
<img src="svg/025_-7f3e9d571a844a11bb63985d3f3c2322ab5ec77807d9a07322b953fc7d6ae1d2.svg?sanitize=true?invert_in_darkmode" align="middle" width="61.96384pt" height="21.090912pt"/> and 
<img src="svg/025_-ff94875a3d92303cbb431299eed6c09c9012ffa946d40f626d6b1c5b752b1f82.svg?sanitize=true?invert_in_darkmode" align="middle" width="67.16841pt" height="21.090912pt"/>, 
<img src="svg/025_-536fd606830ee180d72a16c8926f59cf3f821bc800075bb2481c58c262755c9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="65.54149pt" height="22.72728pt"/> and 
<img src="svg/025_-d5abc6016629ddfc2566f9fca71d3fd71641ae7b81121596d02e1a900aed5763.svg?sanitize=true?invert_in_darkmode" align="middle" width="65.60022pt" height="21.090912pt"/>. 

Let me repeat what we know about 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/>: It should be a vector of length one (because it's a column in a rotation matrix), and 
<img src="svg/025_-ebf37169094cb244327c5e0ca13c585e9b4287704bf18075964d89c7c91375be.svg?sanitize=true?invert_in_darkmode" align="middle" width="64.38058pt" height="21.090912pt"/>, 
<img src="svg/025_-a9ea6e14ef8298f7d82f523257e9125e87ad90e415843152a783504f7826cbf6.svg?sanitize=true?invert_in_darkmode" align="middle" width="63.53203pt" height="21.090912pt"/>, 
<img src="svg/025_-ff94875a3d92303cbb431299eed6c09c9012ffa946d40f626d6b1c5b752b1f82.svg?sanitize=true?invert_in_darkmode" align="middle" width="67.16841pt" height="21.090912pt"/>, that is 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/> is orthogonal to 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> and 
<img src="svg/025_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/> and (due to 
<img src="svg/025_-ebf37169094cb244327c5e0ca13c585e9b4287704bf18075964d89c7c91375be.svg?sanitize=true?invert_in_darkmode" align="middle" width="64.38058pt" height="21.090912pt"/> and both 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/> and 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/> being unit vectors) has an angle with 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>whose cosine equals 1. But then ... 
<img src="svg/025_-631edf7d4d27e6dedc1adb76ad93275a8cf4bc4859b9269f20fca0a3791480b3.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.195118pt" height="14.090904pt"/> is 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>. 

Similarly, 
<img src="svg/025_-d4626957e832cd5ff45c066cddbe369d7a72e3a1582be89c66ee43791018b96f.svg?sanitize=true?invert_in_darkmode" align="middle" width="41.905205pt" height="22.72728pt"/> and 
<img src="svg/025_-243b1067c21770a30178e3ffa7865113ba9d6b407b3480be447643628e6bf03d.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.600323pt" height="14.090904pt"/>. 

Hence the matrix rotating 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>, 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>, 
<img src="svg/025_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/> onto 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> is 


<p align="center"><img src="svg/025_-49d1253b2e22f03133fb845a82e787bc7a160f2c4ba0fd38be30989737fc104d.svg?sanitize=true?invert_in_darkmode" align="middle" width="229.50842pt" height="59.821503pt"/></p>


Obviously, this rotation 
<img src="svg/025_-89bf392c5b21c4672e6ecfa74f684dbb97a5600052fba934205ce8bc2be34b16.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.481813pt" height="22.36361pt"/> reverts the effects of 
<img src="svg/025_-3c208d8b0999e597e95269a25ca4154072533c5424b458951e31db71eee958d4.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.09662pt" height="22.36361pt"/> (first rotating 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/>, 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> onto 
<img src="svg/025_-3d4cfb39a137c5c0483a2a042d9c826e0f82ecfea341ec4951ff1cbe59d241a7.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.912958pt" height="14.090904pt"/>, 
<img src="svg/025_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>, 
<img src="svg/025_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/> and then rotating those back to 
<img src="svg/025_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>,

<img src="svg/025_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/025_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> means everything is back to where it started). 


<img src="svg/025_-89bf392c5b21c4672e6ecfa74f684dbb97a5600052fba934205ce8bc2be34b16.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.481813pt" height="22.36361pt"/> is the so-called "inverse" of 
<img src="svg/025_-3c208d8b0999e597e95269a25ca4154072533c5424b458951e31db71eee958d4.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.09662pt" height="22.36361pt"/>: 
<img src="svg/025_-65fbf09ab0c8a077aeffb9a69ecaa3b45955490234fb84adadf75daf2a6b9b76.svg?sanitize=true?invert_in_darkmode" align="middle" width="68.08206pt" height="27.28537pt"/>. 

So, to recap: For a rotation 
<img src="svg/025_-3c208d8b0999e597e95269a25ca4154072533c5424b458951e31db71eee958d4.svg?sanitize=true?invert_in_darkmode" align="middle" width="17.09662pt" height="22.36361pt"/>: 


<p align="center"><img src="svg/025_-b51da0633e727eabe59b2ab607dcbd1af393648f6284d5c0194999771e6fdec2.svg?sanitize=true?invert_in_darkmode" align="middle" width="73.058136pt" height="14.925764pt"/></p>

 

(For general matrices 
<img src="svg/025_-d5a55d7d8c0bcc7e85aa3889d8b84eaa8f12d4f2e763837c89a5033223623448.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.818218pt" height="22.36361pt"/>, the inverse 
<img src="svg/025_-0eafc42dfbee574c107852fc809d5bb6ae1d84b438466f2c6494be77c5a8172b.svg?sanitize=true?invert_in_darkmode" align="middle" width="33.049248pt" height="27.28537pt"/> is usually more difficult to compute, and for many matrices, it does not even exist.) 

There'd be more to say about matrices and inverse; and even about rotations (for example, we have not talked about the matrix describing the rotation
around a given axis). But I think that's enough "math lecture" for now (and for that example of rotating around a given axis, we can use a specific nalgebra function to help us out).

[Continue](026_Camera.md)
