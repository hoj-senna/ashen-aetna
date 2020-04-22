## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Not so linear

## Translations 

We have just seen how we can so nicely write linear transformations in form of matrices. The problem: Some of the important functions are not linear. 
The worst offender: Shifting (translation) by a constant (non-zero) vector, say: shifting by 
<img src="svg/015_-460db09660bdd394f9138f9755b667912c9d61f76a06e0b0e6f45c0bdffc2360.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/>, that is 


<p align="center"><img src="svg/015_-0111e7c793474c131df1c841a647aec53144cee76e624891c5ee191e56743772.svg?sanitize=true?invert_in_darkmode" align="middle" width="125.44687pt" height="58.909664pt"/></p>

 
Why is that not linear? Well, check the previous chapter for the definition: 
<img src="svg/015_-076aa504bb39d382aeeaa622b051382d2e3fe973b34a439629e4b145cc2e27e2.svg?sanitize=true?invert_in_darkmode" align="middle" width="175.21214pt" height="24.545448pt"/>. That fails, for example for 
<img src="svg/015_-e018fa7fa395f04131e2a3a84884a2a52c2caffbfea94243f88ac5e6d9755767.svg?sanitize=true?invert_in_darkmode" align="middle" width="83.62554pt" height="14.090904pt"/>: 


<p align="center"><img src="svg/015_-6fb75b42272a0d75c77f3c19f5dd78de5c75f98adfd94b9a52bc260cc114eb04.svg?sanitize=true?invert_in_darkmode" align="middle" width="629.3617pt" height="58.909664pt"/></p>

Or, even easier to check: 
<img src="svg/015_-2d927248ee3fc24911d01b70f7417b66ed9a9b4a0c8a8f284e4d128fd636bb79.svg?sanitize=true?invert_in_darkmode" align="middle" width="65.2273pt" height="24.545448pt"/> (where 
<img src="svg/015_-eab005f4b9f5f44ed2e5d8fb5254be5ef60062b8c6675d9f2e1c5b964de951d6.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.727309pt" height="21.090912pt"/> is short for 
<img src="svg/015_-f8245a508a5ff7f828b702cf7b7b5c918f11ee06948bc153645fb471fb785a6f.svg?sanitize=true?invert_in_darkmode" align="middle" width="44.090946pt" height="67.09143pt"/>, and where we note that every linear map sends 
<img src="svg/015_-eab005f4b9f5f44ed2e5d8fb5254be5ef60062b8c6675d9f2e1c5b964de951d6.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.727309pt" height="21.090912pt"/> to 
<img src="svg/015_-eab005f4b9f5f44ed2e5d8fb5254be5ef60062b8c6675d9f2e1c5b964de951d6.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.727309pt" height="21.090912pt"/>).
If this map is not linear, we cannot write it as a matrix. But we want to! (Matrices are well-understood, convenient, there are crates we can use for computations 
involving them ...)

What do we do? We use a clever trick. We use vectors with four components (of course, you could have guessed this: After all, it was the mysterious fourth component 
that set us on our current tangent): quadruplets of numbers would have worked as well as the triplets we used in 
the introduction of vector spaces, and we can easily work with those and matrices of size 4x4 instead of 3x3. And then we write every vector 
<img src="svg/015_-9a3816bce39d45f87605630ade95758219d665a204e436d2c269917cbf57cf84.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261383pt" height="67.09143pt"/> as 


<p align="center"><img src="svg/015_-94e783d1127c64eaa04051d4d3864a926c30e7e3c495ece8417d272e127bd924.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261375pt" height="78.54619pt"/></p>

Yep, always with a 
<img src="svg/015_-f86654b866afd449dbda75632626dd95bf1c2f842ffb147409ab1b0a859b4395.svg?sanitize=true?invert_in_darkmode" align="middle" width="12.727309pt" height="21.090912pt"/> in the last component. 

(Note: the set of all these vectors, 
<img src="svg/015_-a82952f08c0ee46e2b6022e19f7a08ab5b177b572c777d1f3e5e406e252c3e1a.svg?sanitize=true?invert_in_darkmode" align="middle" width="160.64377pt" height="86.72797pt"/> is not a vector space (if we add two of its elements, we leave the set, because the last component becomes different from 1) — but 
that's not a problem. It's a subset of 
<img src="svg/015_-9c16e204a0adc1ea8c7b6d2275a4278f20441d1afe177e86339bcb69660e9621.svg?sanitize=true?invert_in_darkmode" align="middle" width="22.714949pt" height="27.28537pt"/>, and that *is* a vector space.) 

Every 3x3-matrix can be seen as a 4x4 matrix. 

Say, we have 
<img src="svg/015_-4b18c32f2aa2a30f39a03270ce4365f9bfc9918448c92993c61176d2fa8a88b5.svg?sanitize=true?invert_in_darkmode" align="middle" width="163.45882pt" height="67.09143pt"/>. Then the 4x4-matrix is 
<img src="svg/015_-e6c66dbc77c424e3f6dd6a85e0da9b25638af16a1975be0706ad6595f953e734.svg?sanitize=true?invert_in_darkmode" align="middle" width="66.76221pt" height="47.454872pt"/>, which supposed to be short for:


<p align="center"><img src="svg/015_-70da91a9f9343ac1ef06701f502353c84197d949fdc423cfa8a33440256577cc.svg?sanitize=true?invert_in_darkmode" align="middle" width="155.22101pt" height="78.54619pt"/></p>


Let's check what happens if we multiply this by the vector 
<img src="svg/015_-9a3816bce39d45f87605630ade95758219d665a204e436d2c269917cbf57cf84.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261383pt" height="67.09143pt"/>, that is by 
<img src="svg/015_-23bdc492e27539705c5b0a1fd8616a05e0a1757421edab1e14e6983a1d0dad7f.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261383pt" height="86.72797pt"/>:


<p align="center"><img src="svg/015_-39199771732296d001b8d399f1b0063ea2d69d08abec027b6e5e563fc14a1190.svg?sanitize=true?invert_in_darkmode" align="middle" width="676.9076pt" height="78.54619pt"/></p>

Indeed, we obtain that four-component vector which corresponds to the result of 
<img src="svg/015_-3a55f5c8594205871dbd26ced65abfec97772b3fdc00351430acd8f867637e8c.svg?sanitize=true?invert_in_darkmode" align="middle" width="25.337177pt" height="22.36361pt"/> in the three-component setting. 

In other words: We have not lost anything. Has something become better? 

Oh! Have a look at the following: 


<p align="center"><img src="svg/015_-58d5868e0f11523357be0c0438167eacf88ce796ceb74ec6adde515117ab342f.svg?sanitize=true?invert_in_darkmode" align="middle" width="766.4529pt" height="78.8898pt"/></p>

That's a *matrix* describing exactly the shift that we have identified as *not-a-matrix* in the beginning of this chapter! 

As you can imagine, shifting things around (say, moving some 3D model to a different position) is not entirely unimportant in a videogame, for
example. So, this is great and we should start liking 4x4-matrices better than 3x3-matrices and should write points with three coordinates with four
coordinates instead, the last of which is 1.

Why this works, is not so clear from the above. For now, it's a clever hack. We will consider this in more detail in the next chapter. First, let us
have a look at other "not always so linear" maps: 

## Projections

If you want to draw something such that it is visible that there is some depth to it, you may resort to some perspective projection with all lines
that run "into" the image moving closer together and approaching some "vanishing point" on the horizon. You can see a similar effect in photographic
images, too: 
![image of train tracks](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2a/Railroad-Tracks-Perspective.jpg/90px-Railroad-Tracks-Perspective.jpg) 
(with thanks to Wikipedia). See the railroad tracks? See how the railroad's borders come closer together the further "into" the image they move? 

We have a problem with that. To see, why, we have to talk about parallel lines. 

A line through some point 
<img src="svg/015_-7bdb43e8ea115650db03287bebdf6e3858ff4bcb70ca0ac88da12be3e7f29672.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.897743pt" height="14.090904pt"/> with direction (vector) 
<img src="svg/015_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> is the entirety of points that can be reached by adding a multiple of 
<img src="svg/015_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> to 
<img src="svg/015_-7bdb43e8ea115650db03287bebdf6e3858ff4bcb70ca0ac88da12be3e7f29672.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.897743pt" height="14.090904pt"/>, in
symbols:


<p align="center"><img src="svg/015_-b9e80e5ca6b4298521f0231f3b534eabc00db10669a0339daa46a880ed849362.svg?sanitize=true?invert_in_darkmode" align="middle" width="118.78008pt" height="16.363655pt"/></p>

 
or shorter: 
<img src="svg/015_-3557ff3dc8146e9e9cd30afc2b49d06ec32631691b6c2e8233dd185e82108f68.svg?sanitize=true?invert_in_darkmode" align="middle" width="54.23482pt" height="22.545433pt"/>. Note that this representation is not unique: The same line can be written in different ways. For example, if we set 
<img src="svg/015_-64ed5961d04560e8ecda33c67e1fca2e7b7ddcb02694106d6cdf4cf5b2d2a91e.svg?sanitize=true?invert_in_darkmode" align="middle" width="72.84456pt" height="19.09092pt"/> and 
<img src="svg/015_-6c936b4bbd9e9d67764f5dfa0604a71cb9e4c0d5fbafad15fe34e7f9df18443a.svg?sanitize=true?invert_in_darkmode" align="middle" width="55.219654pt" height="21.090912pt"/>, 
then 
<img src="svg/015_-5fdb84235526c4414af37561467dc2fb95fdc49744bd344aa904879499081f59.svg?sanitize=true?invert_in_darkmode" align="middle" width="128.63622pt" height="22.545433pt"/>, even though 
<img src="svg/015_-7bdb43e8ea115650db03287bebdf6e3858ff4bcb70ca0ac88da12be3e7f29672.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.897743pt" height="14.090904pt"/> and 
<img src="svg/015_-a2639d05581ecd60df8267078d3f988185c84b6fead7de2515bc6106bd41ecd4.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.155371pt" height="14.090904pt"/> are different, and so are 
<img src="svg/015_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/> and 
<img src="svg/015_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/>. This claim is not that 
<img src="svg/015_-b0497924df468e74a896ff4aab9fb328e42fa3694e6275083526a02ff1227aa7.svg?sanitize=true?invert_in_darkmode" align="middle" width="124.09078pt" height="22.72728pt"/>; rather, it is that
for every number 
<img src="svg/015_-70fb49c2edd4430279963380fb58a1f42eb56837d0020ea601f5f9dbdacf6d4d.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.090954pt" height="22.72728pt"/> we can find a number 
<img src="svg/015_-64ecaa387b7bc6d5045667f010c7f0368062433b4cefb4c6dcfa27aefc8b5dc4.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.405361pt" height="14.090904pt"/> such that 
<img src="svg/015_-dbbc35aa5f1c4fbdec7dfb192972e6bf3c8a1df170dde3a1bb1b4a01de78bf66.svg?sanitize=true?invert_in_darkmode" align="middle" width="124.40518pt" height="22.72728pt"/>, and vice versa (for every 
<img src="svg/015_-64ecaa387b7bc6d5045667f010c7f0368062433b4cefb4c6dcfa27aefc8b5dc4.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.405361pt" height="14.090904pt"/> some corresponding 
<img src="svg/015_-70fb49c2edd4430279963380fb58a1f42eb56837d0020ea601f5f9dbdacf6d4d.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.090954pt" height="22.72728pt"/>). And that is the case.
(Try it. You should end up with 
<img src="svg/015_-330d52dc77cf6c8e811d54a9658dd96510603dcce56e44ab67f7adbff6ac0a62.svg?sanitize=true?invert_in_darkmode" align="middle" width="61.65754pt" height="29.490143pt"/> and 
<img src="svg/015_-87ed1ae2f5defd4d0c89bc0f2c71b52d9bb20ce7a85402273c7903edebc77ad6.svg?sanitize=true?invert_in_darkmode" align="middle" width="82.132515pt" height="22.72728pt"/>, I hope.) 

Two lines are parallel if they have the same direction. Again, this is not 
<img src="svg/015_-3ac7b2f86b702688ec2d5110663b5aeaa9f7730b8117a2caa66a46a38555a49c.svg?sanitize=true?invert_in_darkmode" align="middle" width="47.037804pt" height="14.090904pt"/> (if the lines are written in the form 
<img src="svg/015_-3557ff3dc8146e9e9cd30afc2b49d06ec32631691b6c2e8233dd185e82108f68.svg?sanitize=true?invert_in_darkmode" align="middle" width="54.23482pt" height="22.545433pt"/> and 
<img src="svg/015_-678ad32c0074bb7a60c0d0d8deffb54dfa8e1794069bfa42509b91c86c4a8ea0.svg?sanitize=true?invert_in_darkmode" align="middle" width="57.128815pt" height="22.545433pt"/>), but

<img src="svg/015_-b4341b0ad706b6b604dfceb81eb5334771d75b59f10a4bdb3b5a486941db32ac.svg?sanitize=true?invert_in_darkmode" align="middle" width="70.674225pt" height="22.545433pt"/>. It boils down to "
<img src="svg/015_-0ada2099e949b8de82c56b5dbf89f07bebfd917eae535265b34faeda1c8ec2e4.svg?sanitize=true?invert_in_darkmode" align="middle" width="16.700802pt" height="14.090904pt"/> is a multiple of 
<img src="svg/015_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>". So, two lines are parallel if (and only if) they can be written in the forms 
<img src="svg/015_-3557ff3dc8146e9e9cd30afc2b49d06ec32631691b6c2e8233dd185e82108f68.svg?sanitize=true?invert_in_darkmode" align="middle" width="54.23482pt" height="22.545433pt"/> and 
<img src="svg/015_-0ac1ba7c6c71985e7e53ce621d8beceb1bbf23f02ebebb84009c2cfef3aad2a3.svg?sanitize=true?invert_in_darkmode" align="middle" width="53.49244pt" height="22.545433pt"/>, with
possibly different 
<img src="svg/015_-7bdb43e8ea115650db03287bebdf6e3858ff4bcb70ca0ac88da12be3e7f29672.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.897743pt" height="14.090904pt"/> and 
<img src="svg/015_-a2639d05581ecd60df8267078d3f988185c84b6fead7de2515bc6106bd41ecd4.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.155371pt" height="14.090904pt"/>, but the same 
<img src="svg/015_-6e7c43170fefd2ba7da09b4f17c3509e40a2cad90b83155cf4313c2b093a2d02.svg?sanitize=true?invert_in_darkmode" align="middle" width="13.064421pt" height="14.090904pt"/>.

Now let us look at what a linear transformation — say, 
<img src="svg/015_-ba2cbb73b8ff1cf346c8d7b7fb0331e62471f2af60d80ae909030fb1f62f20da.svg?sanitize=true?invert_in_darkmode" align="middle" width="14.318251pt" height="22.72728pt"/> — does with parallel lines (
<img src="svg/015_-3557ff3dc8146e9e9cd30afc2b49d06ec32631691b6c2e8233dd185e82108f68.svg?sanitize=true?invert_in_darkmode" align="middle" width="54.23482pt" height="22.545433pt"/> and 
<img src="svg/015_-0ac1ba7c6c71985e7e53ce621d8beceb1bbf23f02ebebb84009c2cfef3aad2a3.svg?sanitize=true?invert_in_darkmode" align="middle" width="53.49244pt" height="22.545433pt"/>). By its linearity, 


<p align="center"><img src="svg/015_-6ba423f503a73427dd7ca88b035b7210a8efcbda13708214ff8895d1a78a9e7d.svg?sanitize=true?invert_in_darkmode" align="middle" width="302.93192pt" height="16.363632pt"/></p>

that is 
<img src="svg/015_-84137311651f711419470abbd4716cb2011344bac7b2d23ad9af40c02f7601bc.svg?sanitize=true?invert_in_darkmode" align="middle" width="193.24246pt" height="24.545448pt"/>. In the same way, 
<img src="svg/015_-747166a2bdfe737ed0c3baa2b769bbbc4c58810d5d945a02d5c32d4e28f04e3e.svg?sanitize=true?invert_in_darkmode" align="middle" width="191.7577pt" height="24.545448pt"/>. Comparing the directions 
<img src="svg/015_-2278ce831ba2a19ec2d006ab58f07261678cf0b3a87f8bbc3b799f0f6affd0a4.svg?sanitize=true?invert_in_darkmode" align="middle" width="35.564495pt" height="24.545448pt"/> and 
<img src="svg/015_-2278ce831ba2a19ec2d006ab58f07261678cf0b3a87f8bbc3b799f0f6affd0a4.svg?sanitize=true?invert_in_darkmode" align="middle" width="35.564495pt" height="24.545448pt"/> of these results, we note:
Transforming parallel lines by a linear map results in parallel lines. 

Going back to the picture: What were clearly parallel lines in reality was turned into clearly non-parallel lines in the photograph. (Lines from
approximately the bottom left corner to some point, and from the bottom right corner to the same point. They do not have the same direction.) 

That means: Whatever transformation has taken place to turn reality into a photo, it was not a linear map. What a pity. Taking pictures in the same
way as a camera sounds like it would be useful for somewhat realistic ways of turning virtual scenes into images on a screen. 

Not to be too unfair to projections: There are projections that are linear. For example the one turning 
<img src="svg/015_-9a3816bce39d45f87605630ade95758219d665a204e436d2c269917cbf57cf84.svg?sanitize=true?invert_in_darkmode" align="middle" width="45.261383pt" height="67.09143pt"/> into a point 
<img src="svg/015_-1aff31957ce02b31e9e54812d7f2fc14522f63a18b1143a09b418f56b01eacc9.svg?sanitize=true?invert_in_darkmode" align="middle" width="40.715946pt" height="47.454872pt"/> by
simply ignoring the third component (this was how points in our "box-like" kind-of-screen were interpreted as points on the screen when we were last
dealing with graphics and not only with hopefully helpful theory). 

But the camera's perspective projection? Nope, it isn't. Fortunately for us, we do not even need a *new* trick to rectify this problem. 


Let us try to better understand what is going on with this miraculous additional component in our vectors.

[Continue](016_Homogeneous_Coordinates.md)
