## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Homogeneous coordinates

In the previous chapter we have extended vectors with three components into vectors with four components by adding an additional component of value 1.
This was, somehow, helpful for turning translations into *linear* maps (and I have hinted that it may also be useful in connection with perspective
projections) — but we did not look into *why* this works or what is really going on. 

I would like to illustrate the process by some pictures. We went from a three- into a four-dimensional space, and that is hard to draw (or follow, for
that matter). Therefore, let's instead go from one- to two-dimensional spaces. Yep, we're starting from vectors with one component. That is: from
numbers.

Here they are: 


<p align="center"><img src="svg/016_-0b1d23ed6d4f8ce31de2c664b784df7279a61980661af3505fdbe03a13b5ce17.svg?sanitize=true?invert_in_darkmode" align=middle width=298.2402pt height=21.914284pt/></p>


These are the "points" 
<img src="svg/016_-5253f52fc2bafa7ce1da5ce4f7ba3787e3e03a3d48738ddfd0a9aafea2fac2c2.svg?sanitize=true?invert_in_darkmode" align=middle width=94.545364pt height=21.090912pt/>. We embed this line into a plane:



<p align="center"><img src="svg/016_-03ab3f11a7110ead4da596aaa066be495d58c84ad05c82df10fdfd7e5839f289.svg?sanitize=true?invert_in_darkmode" align=middle width=298.83945pt height=96.73755pt/></p>


The origin of this plane is somewhere outside the line (thick point); I have moved it so that the additional component is just the 1 we had added previously. 
The meaning of the two basis vectors 
<img src="svg/016_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align=middle width=19.315155pt height=14.090904pt/> and 
<img src="svg/016_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align=middle width=18.872374pt height=14.090904pt/> in this new plane: 
<img src="svg/016_-fc52642d0576571514b0c0bde5e9bbbc701c8c36cc33946b8cf9563f138d5616.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> describes how to get from one point to the other, so it encodes
direction and scale of the line; 
<img src="svg/016_-4de52a7a1a2ff4ff362dd0b6bec6b8579043ee90177cdb8c803acb57e2d95b07.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> answers: Where is our line? (Or, more precisely: the 0 on it.) 

Now let us try to see how we can describe a translation. We want to shift all numbers (that is: all points on our line) to the right, by 
<img src="svg/016_-ef5db0f489b15cbf3832662f168e42c98e00b502f0819c4453853d40d8f746b5.svg?sanitize=true?invert_in_darkmode" align=middle width=10.454576pt height=20.129883pt/>.

What happens? Did we change the direction or scaling of our line? No. So, 
<img src="svg/016_-fc52642d0576571514b0c0bde5e9bbbc701c8c36cc33946b8cf9563f138d5616.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> remains 
<img src="svg/016_-fc52642d0576571514b0c0bde5e9bbbc701c8c36cc33946b8cf9563f138d5616.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/>.

If we can make this fnuction into a matrix (of size 2x2, because the vectors have two components), that means it should look like this: 


<p align="center"><img src="svg/016_-7628f8a54dbc0058bc860efb008b6eb6d6b17bee04b58b795fa88f6cde816aab.svg?sanitize=true?invert_in_darkmode" align=middle width=96.30757pt height=39.273117pt/></p>


What happens with 
<img src="svg/016_-4de52a7a1a2ff4ff362dd0b6bec6b8579043ee90177cdb8c803acb57e2d95b07.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/>? Well, we wanted to shift it by 
<img src="svg/016_-ef5db0f489b15cbf3832662f168e42c98e00b502f0819c4453853d40d8f746b5.svg?sanitize=true?invert_in_darkmode" align=middle width=10.454576pt height=20.129883pt/>. It should, therefore, become 
<img src="svg/016_-bee50bd6572a21e8b01322a9bade78364fccee0dd83bb46d9cab5de47b057298.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/>. And thus we fill in the missing pieces: 


<p align="center"><img src="svg/016_-7ea0ceb1a247e88fbfcd6c5ee68ac4c605a7746ca4a9a7b5887709a0df721b5b.svg?sanitize=true?invert_in_darkmode" align=middle width=89.489426pt height=39.273117pt/></p>


If we try to visualize this transform, we might end up with pictures like the following (for 
<img src="svg/016_-272579a96d8e7a9ba913160df4169d5a14595c43a112c307f03cb06f688d97b2.svg?sanitize=true?invert_in_darkmode" align=middle width=40.45448pt height=21.090912pt/>): 


<p align="center"><img src="svg/016_-085866c5459795735a71d8d502ea67df0b0ffc258422026dd54fd0c22969f695.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/></p>

to 


<p align="center"><img src="svg/016_-5d7ea56864a14fcd660f8c1c9ed475ea7ebfa5e7812a181224d0910cbc0d17ff.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/></p>

(where for every point 
<img src="svg/016_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align=middle width=13.064421pt height=14.090904pt/> in the first picture, a point of the same colour has been drawn at 
<img src="svg/016_-3a55f5c8594205871dbd26ced65abfec97772b3fdc00351430acd8f867637e8c.svg?sanitize=true?invert_in_darkmode" align=middle width=25.337177pt height=22.36361pt/> in the second.) 

This is not so much a translation, which would rather have resulted in this: 


<p align="center"><img src="svg/016_-b66ea9ffbc6157bbb6aad0cfe132914d348e35f646091bfe8748904f46d6a1ee.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/></p>

 — but rather a *shearing*. 

Restricted to the single line of interest (see the yellow line in the pictures), however, it acts the same. (And it should be noted that every
translation on this line can be extended to a shear transform of the plane — and those *are* linear. This is how we transformed the translations into
a linear function.) 

Points outside our single lucky line, however, feel the difference between shift and shear. (Pictures: original (gray), sheared (blue), shifted (violet))


<p align="center"><img src="svg/016_-3892faf4b8d4960c97b934e3927693cc6ec01530819e0c5e75169599abc5ecff.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/>
<img src="svg/016_-2fc7805ef993376777f6c2f70bfc99ed00cd0250935ca12744e727d610ed9517.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/>
<img src="svg/016_-21041c34fc3893f557b6d023a6782f62b6a80f7627b39fffc4286c3641be263f.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/></p>

Or the points in one system: 


<p align="center"><img src="svg/016_-700631fe95fe3b6c9222a9d150f7cf90cc6196bf09ebdafd4d7e9801dd5cc241.svg?sanitize=true?invert_in_darkmode" align=middle width=285.1803pt height=276.67883pt/></p>

While the (blue and violet) circles end up on top of each other, the crosses differ.



Where 
<img src="svg/016_-4de52a7a1a2ff4ff362dd0b6bec6b8579043ee90177cdb8c803acb57e2d95b07.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> was moved to 
<img src="svg/016_-c303fca2f60dba1a3508d947ee8db3bac69e36f16a5aa64edf92c4764a27d702.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/>, 
<img src="svg/016_-439ad61faca450e988a906775f4b752a57d009f277cb8b980ed7610997802c3d.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> ended up in 
<img src="svg/016_-439ad61faca450e988a906775f4b752a57d009f277cb8b980ed7610997802c3d.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/> (not in 
<img src="svg/016_-ecadc31848341135fc82970b57defe673e2414cdf55b47b1e77dee89d2aba925.svg?sanitize=true?invert_in_darkmode" align=middle width=39.54551pt height=47.454872pt/>). Is there a way to interpret this? Well,
let's have a look (only the results of the shearing): 



<p align="center"><img src="svg/016_-01becacad2eebb60d861c274a4f895cdac91e17f3060aa537177fcb8d650dbb7.svg?sanitize=true?invert_in_darkmode" align=middle width=285.47873pt height=276.67883pt/></p>


Indeed, it seems that the two points lay on one line (through the origin) before being transformed (i.e.: the red points), and that also after the
transformation both points (i.e.: now the blue points) lie on the same line (through the origin): 



<p align="center"><img src="svg/016_-a4529c2b326d59a4f4c90e9a5768fdc0b9797e58643281036a96a58abbc0aeea.svg?sanitize=true?invert_in_darkmode" align=middle width=285.47873pt height=276.67883pt/></p>


(This is not accidental. If 
<img src="svg/016_-bbcecee7d9077169091b378d17ef7ea240461e20ed6238f8ede3656950eb283f.svg?sanitize=true?invert_in_darkmode" align=middle width=56.583305pt height=22.72728pt/>, then 
<img src="svg/016_-f315f3c28e1f7cbabdebc20e5f9d1c52a4478d92496e39f18a1a39760fa41c97.svg?sanitize=true?invert_in_darkmode" align=middle width=171.46599pt height=24.545448pt/>; that is: the transform of 
<img src="svg/016_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align=middle width=16.700802pt height=14.090904pt/> and of 
<img src="svg/016_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align=middle width=13.064421pt height=14.090904pt/> are multiples of each other whenever 
<img src="svg/016_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align=middle width=16.700802pt height=14.090904pt/> and 
<img src="svg/016_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align=middle width=13.064421pt height=14.090904pt/> are — for every linear
map A.) 

Now, one can make the case that we should not distinguish between the two red points (or the other points on the red line, for that matter): Was it
not a very arbitrary choice that we picked the upper one? We could have taken the lower line (with the lower red and blue point) as our representation
of the numbers instead. 



<p align="center"><img src="svg/016_-0f59455fe25b6e1d7ece4a3a4bd360cb429d9d1354c2c0c3f2530f65391c0e6d.svg?sanitize=true?invert_in_darkmode" align=middle width=545.44995pt height=221.40219pt/></p>


That would still go well with our transformation: Whether we claim to map the red point onto the blue point or the red line onto the blue line —
that's not really different.

Let's make it official: All points lying on the same line through the origin (with the exception of the origin) are the same, from now on. What does
that mean in coordinates? That 
<img src="svg/016_-a8c488e62274bf4a37706b411bf20b44cf288e140761b8b360b46e38de104b1f.svg?sanitize=true?invert_in_darkmode" align=middle width=40.01333pt height=47.454872pt/> and 
<img src="svg/016_-1d3697499e090d35726487d41269b8e8f3057a7c29e8e2df05a4feebf0801e4e.svg?sanitize=true?invert_in_darkmode" align=middle width=39.88073pt height=47.454872pt/> are "the same" iff they lie on the same line that is iff there is 
<img src="svg/016_-70fb49c2edd4430279963380fb58a1f42eb56837d0020ea601f5f9dbdacf6d4d.svg?sanitize=true?invert_in_darkmode" align=middle width=14.090954pt height=22.72728pt/> such that

<img src="svg/016_-80579181572ad40b543f47db089dcc859821e9d41b06eb43b2d473a44e0cc12a.svg?sanitize=true?invert_in_darkmode" align=middle width=106.71213pt height=47.454872pt/>. Or, for 
<img src="svg/016_-daeeae27ff0118d6b8d976ad9576b15116ae41b66c4c1698dd338203e242e7b9.svg?sanitize=true?invert_in_darkmode" align=middle width=41.568092pt height=22.72728pt/>: 
<img src="svg/016_-f591e0df6de3a3078e2ddec27cdd9d0518391922f07b14f4444c46cf89377f7d.svg?sanitize=true?invert_in_darkmode" align=middle width=106.71213pt height=47.454872pt/>, which works for 
<img src="svg/016_-f288434485d3e0d5ac6804d3286dc29f735832b09e2727e2af1184813e196e27.svg?sanitize=true?invert_in_darkmode" align=middle width=44.23826pt height=28.294647pt/>. Hence: 
<img src="svg/016_-b4de162ecf95918763cede22167f65ca8d115268faeca2b5a903a56a0c6efe17.svg?sanitize=true?invert_in_darkmode" align=middle width=43.342422pt height=23.180466pt/> or: 


<p align="center"><img src="svg/016_-9db4c4ce680e38fb04bca52090954a83332b6f28a5a82d00c1a79de7a4c03cb3.svg?sanitize=true?invert_in_darkmode" align=middle width=195.91248pt height=39.273117pt/></p>

(Do you remember, how in [Chapter 13](013_Coordinates.md), the two points 
```glsl
	    gl_Position = vec4(0.8,0.4,0.0,2.0);
```
and
```glsl
	    gl_Position = vec4(0.4,0.2,0.0,1.0);
```
coincided? That is the same principle!) 


There is indeed a "real world" setting where identification of all points on a line is natural: Perspective projection. Imagine your eye (or a camera)
in the origin. Then all points on the blue line lie in the same direction and cannot be distinguished. (Or rather: you end up seeing only one of those
points: the closest one where there is something to see and not just (transparent) air.) 

That's why these coordinates also fit so well with the perspective projection mentioned in the previous chapter. We will, at some later time, probably
return to projections, so let's not go into more detail here. 

But I think, by now we have seen where this fourth component in coordinates in our shader comes from and, roughly, what it means. 

Let us return to our shader.

[Continue]  ()
