## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Vectors, linear maps and matrices

## Vectors 

The mathematical concept of choice when dealing with points and directions in space is that of a **vector**. 

What is a vector? 

The formally correct answer may be "an element of a vector space" and provoke the question "and what's a vector space?", but in the end it boils down to "something that
you can multiply by numbers and that you can add to other vectors" (footnote: where this multiplication and addition work as one could reasonably
expect). Example ("something" admittedly is not very helpful in getting a feeling for them): triplets of numbers, like 



<p align="center"><img src="svg/014_-717f59e91d8bb6c9a0d16cde7c49ffc01ad203e659c1abb041dc62bf3e80012b.svg?sanitize=true?invert_in_darkmode" align="middle" width="62.272865pt" height="58.909664pt"/></p>


or, more precisely, all of the following 


<p align="center"><img src="svg/014_-091f4a40ee0f5ae35a3ec01271f9a0cbf7db230b99a774ccfa584840f49993c0.svg?sanitize=true?invert_in_darkmode" align="middle" width="160.64368pt" height="58.909676pt"/></p>


Adding two of these, or multiplying with numbers ... let's see. How about 



<p align="center"><img src="svg/014_-08aae29ab4513dc2089a15fc5f0774374599f43d3b1fa72ac5fb1f3b5e5e2e74.svg?sanitize=true?invert_in_darkmode" align="middle" width="213.53009pt" height="58.909664pt"/></p>

and


<p align="center"><img src="svg/014_-5c706d3ee88c6e579f7c98a9f029dc2da529cbec695fa76bba2469a1d300a8a0.svg?sanitize=true?invert_in_darkmode" align="middle" width="127.34091pt" height="58.909664pt"/></p>


That works for me, and since the rules are based on the usual addition and multiplication in 
<img src="svg/014_-f8c4f7be475a41c22dab878a3cb8e3a28b5b9b934120c0a830a3691ec17d2a72.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.363674pt" height="22.545433pt"/>, we can be confident that all additional
requirements from the footnote that was no footnote are satisfied. (We could also check this, but then I'd have to elaborate on what "work as one
could expect" means, and I'd rather avoid talking about the [definition of vector spaces](https://en.wikipedia.org/wiki/Vector_space#Definition) in more detail.) 

This definition also has the effect that we can split every vector in the following way: 


<p align="center"><img src="svg/014_-9443c9c919619aee52cb037048bd886b1175d87df91c4fe2cdc34245e66e04bd.svg?sanitize=true?invert_in_darkmode" align="middle" width="244.73474pt" height="58.909664pt"/></p>

or 


<p align="center"><img src="svg/014_-c3ae70d2b748d48c222fab9c3ad1852212f850deace3529b163e38bc4bf7cd9f.svg?sanitize=true?invert_in_darkmode" align="middle" width="170.72218pt" height="58.909664pt"/></p>

if we introduce new names 
<img src="svg/014_-515c162df8068444131fc2014ca56cd7af00780b9f4c8abfd42003e4613ef4aa.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.69867pt" height="67.09143pt"/>, 
<img src="svg/014_-f3e37fb3b86915f3b85ec8c901f1e875d1bdfa108b2f20cba99ff93573ce62ba.svg?sanitize=true?invert_in_darkmode" align="middle" width="78.25589pt" height="67.09143pt"/>, 
<img src="svg/014_-728755c6a7d1733a9272d45131f74b22d9743c6273619e654db740d665e3dc89.svg?sanitize=true?invert_in_darkmode" align="middle" width="77.961845pt" height="67.09143pt"/>. 

Okay, so, for example those triplets of numbers are vectors. 

Any other meaning? For example something less abstract? 

Sure. How about motions in space?

Think of *shifting* ("translations", not in the language sense), and let's say 
<img src="svg/014_-460db09660bdd394f9138f9755b667912c9d61f76a06e0b0e6f45c0bdffc2360.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/> means moving something by one metre to the right, two metres down and three metres forward. Can we
multiply a shift by a number? Sure: times 3 means thrice as far. Can we add two shifts? Imagine you carry out two shifting motions after each other.
Then you realize you could have achieved the same by just one (different) shift. Well, let's call that one the sum of the previous two.
(Actually, my description of "moving by one metre to the right, two metres down *and* three metres forward" seems to have this as a builtin feature.)
So these shifting motions form another vector space; 
<img src="svg/014_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/014_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/>, 
<img src="svg/014_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> are motions to the right, down or forward, respectively, and we can describe
every shifting as sum of multiples of these. 

In a similar vein, vectors are often visualized as "arrows" (where two arrows are considered the same if they have the same length and direction, no
matter whether they start at the same point), and there are also many physical quantities that are best encoded by vectors (velocities (with their
magnitudes and directions), to name but one of the most prominent examples). 

What about points? Our goal was to define points. Do points also form a vector space? 

Welllllll, it is not so clear what "adding two points" should mean. So: kind-of "no"? (At least as long we stick to addition with obvious meanings.) 

But if we designate one special point (the "origin"), we can with reasonable soundness claim that "a point" and "the shift motion needed to get from
the origin to this point" (or "the arrow from origin to this point") are interchangeable. Using the corresponding additions and multiplications by
numbers, we can give meaning to "point plus point" — and very soon claim that *obviously*, you can add two points — no matter how meaningless such an
operation is (and no matter that it would make much more sense to speak of addition of point and translation/arrow than of point and point). "Two
times some point" is just "the point that is twice as far away from the origin, but in the same direction". It is usually not seen as "worth it" to
maintain a distinction between points and "arrows" ("vectors"). In that sense: The points in our screen are vectors, we write them with their three
components. 

## Linear Maps, Matrices 

Now that we have points, let's see how to transform one point into another. "Transform"? Well, somehow take one point 
<img src="svg/014_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>, do something, obtain a
(probably different) point 
<img src="svg/014_-2278ce831ba2a19ec2d006ab58f07261678cf0b3a87f8bbc3b799f0f6affd0a4.svg?sanitize=true?invert_in_darkmode" align="middle" width="35.564495pt" height="24.545448pt"/>. And we heavily restrict the "something" in that description, just so we end up with a well-understood class of transformations that
is extremely easy to work with and that comes with helpful formalism: Linear transformations. 

What are linear functions? Linearity means they work
well together with the two operations we needed to define a vector space: addition of two vectors, and multiplication of number and vector. 

"Work well"
is supposed to mean: the order (first apply the function, then multiply by number versus first multiply, then apply the function) does not matter: 


<p align="center"><img src="svg/014_-be9a9a6fd0ef6824b493c6f5ab93cd3430569668188c4350f77cffcbe8ae292c.svg?sanitize=true?invert_in_darkmode" align="middle" width="365.4322pt" height="16.363632pt"/></p>

 
(for every possible choice of numbers λ and vectors v and w). 

Not every function is linear, but this is still a very large and important amount. (And
there are techniques to pretend that other functions are "locally" and/or "approximately" linear — "derivative" should be the key word here, which
does not concern us for now.) 

One example would be "move each point to the middle between itself and the origin" or "rotate the (each) point around the z-axis by a quarter of a
circle, counterclockwise if we're looking toward positive z". So, scaling, and rotations are linear. These descriptions can become wordy. Fortunately, they are not necessary. 

First observation: When we know what a linear transformation does with 
<img src="svg/014_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/>, 
<img src="svg/014_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> and 
<img src="svg/014_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/>, that's enough information. 

Because we can write every vector 
<img src="svg/014_-6e411caf1456628e381ca94537434bde8603859c2e8aa60741f2452771593a9d.svg?sanitize=true?invert_in_darkmode" align="middle" width="72.87116pt" height="67.09143pt"/> as 
<img src="svg/014_-8036e90f350e82849c51c3bad12dc4615b4ba64380ded82c5282d9a161e732a3.svg?sanitize=true?invert_in_darkmode" align="middle" width="205.60464pt" height="67.09143pt"/> and because 
<img src="svg/014_-ba2cbb73b8ff1cf346c8d7b7fb0331e62471f2af60d80ae909030fb1f62f20da.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.318251pt" height="22.72728pt"/> is linear, we obtain 


<p align="center"><img src="svg/014_-9e1dd4562c57d7b14af746990eb864734b2b880b75427bf788f44395bc524219.svg?sanitize=true?invert_in_darkmode" align="middle" width="593.5072pt" height="17.051868pt"/></p>

Let's say, 
<img src="svg/014_-860738aa2189d56f53ec9d2dcf4ab11f8850ec3ca82f99cfe015638c27d4df4d.svg?sanitize=true?invert_in_darkmode" align="middle" width="101.66658pt" height="67.09143pt"/>, 
<img src="svg/014_-13e2b32205a1a45399e2e2b0290883624ffa7fe3b23e4548d6f9283b2e3bb6bd.svg?sanitize=true?invert_in_darkmode" align="middle" width="102.34692pt" height="67.09143pt"/>, 
<img src="svg/014_-24f39435b8a48ac620b576d59cf532224390eff9e0f5264b79bbdf46fa9359cf.svg?sanitize=true?invert_in_darkmode" align="middle" width="101.708145pt" height="67.09143pt"/> — or, if you prefer a longer form with less line height:

<img src="svg/014_-1bf8836defcc75fc805c0c3f90636d13a148932ba38d95b46c5967a5d0b4a13e.svg?sanitize=true?invert_in_darkmode" align="middle" width="171.75803pt" height="24.545448pt"/>, 
<img src="svg/014_-e0b936e52548750756aa6585854c61c7706762fa7ff59c3f0720daa0af67aefe.svg?sanitize=true?invert_in_darkmode" align="middle" width="174.47058pt" height="24.545448pt"/>, 
<img src="svg/014_-1930dd77f7bee765831be22f7331fd94a52444e7e4be647745a2d9cf1062dc34.svg?sanitize=true?invert_in_darkmode" align="middle" width="171.72495pt" height="24.545448pt"/>. 

Nine numbers to describe the map, or one table of numbers, one "matrix": 


<p align="center"><img src="svg/014_-1e611e284bd86e1d5a5f4306b6b11435ee6f1a9f733ec8e76c111f4942c33d2d.svg?sanitize=true?invert_in_darkmode" align="middle" width="127.7383pt" height="58.909664pt"/></p>

We define the product of the matrix 
<img src="svg/014_-d5a55d7d8c0bcc7e85aa3889d8b84eaa8f12d4f2e763837c89a5033223623448.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.818218pt" height="22.36361pt"/> and the vector 
<img src="svg/014_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> as follows: 


<p align="center"><img src="svg/014_-69a12d82e4f32be59f04e134ed9b90f78a1be7621a026a3328634ff0fde7f9ad.svg?sanitize=true?invert_in_darkmode" align="middle" width="731.1982pt" height="58.909664pt"/></p>

so that 
<img src="svg/014_-3a55f5c8594205871dbd26ced65abfec97772b3fdc00351430acd8f867637e8c.svg?sanitize=true?invert_in_darkmode" align="middle" width="25.337177pt" height="22.36361pt"/> and 
<img src="svg/014_-2278ce831ba2a19ec2d006ab58f07261678cf0b3a87f8bbc3b799f0f6affd0a4.svg?sanitize=true?invert_in_darkmode" align="middle" width="35.564495pt" height="24.545448pt"/> actually coincide.

Trying to put our examples into this framework then looks as follows: 
"Move each point to the middle between itself and the origin" (a.k.a. make everything smaller): 
<img src="svg/014_-7ebd5ee30f20b5a35d5b228430f36f01085e65cd92a2fd2f69889761f8cc21f1.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/> is turned into 
<img src="svg/014_-ede36d978a0b055faac4f523e06dcf748a1f83332dfc86f865be5b77b352015a.svg?sanitize=true?invert_in_darkmode" align="middle" width="56.818233pt" height="67.09143pt"/>, so 


<p align="center"><img src="svg/014_-6884c553d1269846c8c46b73cccaa937f0548910245e4a04a92567bc3a10caab.svg?sanitize=true?invert_in_darkmode" align="middle" width="136.25156pt" height="58.909664pt"/></p>

 

<img src="svg/014_-cfeabf324c44555dcf41d16be7aa376bb426adfc37b4d7f4fb499f78c160124c.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/> becomes 
<img src="svg/014_-4b7d49baace6cfbe53bbdca54c8e20a2cba316685d1fc14915ae29770cc40371.svg?sanitize=true?invert_in_darkmode" align="middle" width="56.818233pt" height="67.09143pt"/>, hence 


<p align="center"><img src="svg/014_-86fda97c0a506b104b49fa6fca663650b965c3bf2f55f5f5e3ce193327436b03.svg?sanitize=true?invert_in_darkmode" align="middle" width="149.43338pt" height="58.909664pt"/></p>

and with 
<img src="svg/014_-02078dbc2d8cb8dc7a281535c130bb37e051a0c77a85c429e16acf67f1740095.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/> similarly being mapped to 
<img src="svg/014_-c1ea4448be8a3de88e31a14d25e3c77a14c710cf18a174a078b86373df8c1b51.svg?sanitize=true?invert_in_darkmode" align="middle" width="56.818233pt" height="67.09143pt"/>, we end up with 


<p align="center"><img src="svg/014_-b7b1e2d79d8f03010adc72e6fc12fc12f8ecba8b24047f2a4115705de46589fc.svg?sanitize=true?invert_in_darkmode" align="middle" width="162.61519pt" height="58.909664pt"/></p>


"rotate the (each) point around the z-axis by a quarter of a circle, counterclockwise if we're looking toward positive z": Rotating about z means that 
<img src="svg/014_-9a4091efeb8864289313155d5be823b1d9aac99a21377bf09b092fc2dae3ae6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.578333pt" height="14.090904pt"/> is fixed (mapped to 
<img src="svg/014_-443cf958b62507f0b46bea119000e2b1c6636eea42e6b7d02bc6d84ebb6f36a5.svg?sanitize=true?invert_in_darkmode" align="middle" width="113.71469pt" height="21.090912pt"/>), hence we can easily begin filling in the matrix: 


<p align="center"><img src="svg/014_-46e066dadb9681df30e8445a6a2fad52ed2066f7424f07e66757e99d94a857ff.svg?sanitize=true?invert_in_darkmode" align="middle" width="123.52426pt" height="58.909664pt"/></p>

 
and we observe that 
<img src="svg/014_-e4a6be150ed064f8efea162de4c63162dad79e434813ba393dbee1f871813e9c.svg?sanitize=true?invert_in_darkmode" align="middle" width="19.315155pt" height="14.090904pt"/> (initially pointing to the right) after turning becomes 
<img src="svg/014_-a0f6309d18cf118ef6292bf726c2c8749a9618695de20799fe01c6d8bf873c6d.svg?sanitize=true?invert_in_darkmode" align="middle" width="38.872276pt" height="19.09092pt"/> (pointing towards the top)


<p align="center"><img src="svg/014_-1f0264d6d04ebca37d88475717a6804db435c34e4b4f0646d453bd409b6449cd.svg?sanitize=true?invert_in_darkmode" align="middle" width="136.70612pt" height="58.909664pt"/></p>

 
And 
<img src="svg/014_-4cb3e80141f893b3ba9d26397b6091217c5798c340ff3b134f2f98f3327f91fa.svg?sanitize=true?invert_in_darkmode" align="middle" width="18.872374pt" height="14.090904pt"/> (originally pointing down)? After rotation it points to the right: 
<img src="svg/014_-439db7401f3eeaac7671b7053eeb0a2635aef881965921d82882bf74e2cc50ac.svg?sanitize=true?invert_in_darkmode" align="middle" width="68.48007pt" height="22.36361pt"/>, that is 


<p align="center"><img src="svg/014_-e3724c25e9d78b1c8ea1ede5fd91c31f5dd1231f335e3d6b9a6a6641c3bcdd2d.svg?sanitize=true?invert_in_darkmode" align="middle" width="137.16063pt" height="58.909664pt"/></p>

 
If we want to rotate, for example, 
<img src="svg/014_-2ab4b2df869d2e4f53db0d9a779f8cf553294068fc0289ba743611d4f67dfbcf.svg?sanitize=true?invert_in_darkmode" align="middle" width="56.818256pt" height="67.09143pt"/>, we compute 


<p align="center"><img src="svg/014_-4491359f48281cef9fd6cf21ee85bd7671bcd44efbb79d35e057eed820bef3fe.svg?sanitize=true?invert_in_darkmode" align="middle" width="499.43304pt" height="58.909664pt"/></p>

 


Bad news: Not all maps are linear, and there are important cases we are missing: 

[Continue](015_Not_so_linear.md)
