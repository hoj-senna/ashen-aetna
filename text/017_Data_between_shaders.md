## Ashen Aetna 
#### — Rustily stumbling around on an ash-covered volcano 
###### (A tutorial on/in/about/with 3D graphics, Rust, Vulkan, ash)

# Passing data between shaders

The point we have been drawing is red. We had hardcoded this colour in the fragment shader: 
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	void main(){
		theColour= vec4(1.0,0.0,0.0,1.0);
	}
```

Of course, we could just change it here if we were interested in different colours. Try 

```glsl
		theColour= vec4(1.0,0.5,0.0,1.0);
```
or 
```glsl
		theColour= vec4(0.0,0.0,1.0,1.0);
```
for example. But it may be more interesting to see how we could get different colours at different points. Or, more generally, how we can get data to
the fragment shader without hardcoding the values. 

One source for data for the fragment shader could be ... the vertex shader, seeing that it runs more or less directly before the fragment shader is
run. 

We have to tell both shaders about our plan: The fragment shader that some value is coming in: 

```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	layout (location=0) in vec4 data_from_the_vertexshader;

	void main(){
		theColour= vec4(1.0,0.0,0.0,1.0);
	}
```
and maybe also what to do with it (use it instead of the value red we had used before): 
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	layout (location=0) in vec4 data_from_the_vertexshader;

	void main(){
		theColour= data_from_the_vertexshader;
	}
```
and the vertex shader has to be told to spit out this data (and we have to give it some value): 

```glsl
	#version 450

	layout (location=0) out vec4 data_from_the_vertexshader;

	void main() {
	    gl_PointSize=10.0;
	    gl_Position = vec4(0.4,0.2,0.0,1.0);
	    data_from_the_vertexshader=vec4(0.0,0.6,1.0,1.0);
	}
```

Okay, the point is no longer red. It has worked.  

Let's go over the code we added: There was `vec4` as data type, okay. Then there were the rather obvious keywords `in` for data being passed into the
shader and `out` for, well, output.  

What about the *variable names*? I mean, we called the variables `data_from_the_vertexshader` in both shaders. Was that necessary? 

Let's try. We keep the fragment shader, but change the vertex shader as follows:
```glsl
#version 450

layout (location=0) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_PointSize=10.0;
    gl_Position = vec4(0.4,0.2,0.0,1.0);
    colourdata_for_the_fragmentshader=vec4(1.0,0.6,1.0,1.0);
}
```

Does it still work? No error messages from the validation layers? Very good. 

Note: *It is not the names of the variables by which they are identified in a different shader*.

And this is where the meaning of `layout (location=0)` plays its role. Let's change it: 

```glsl
#version 450

layout (location=1) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_PointSize=10.0;
    gl_Position = vec4(0.4,0.2,0.0,1.0);
    colourdata_for_the_fragmentshader=vec4(1.0,0.6,0.0,1.0);
}
```

Everything is black. (And it's not because I have changed the colour again, which still is very non-black, according to the numbers. Thank you very
much.) 

But the fix for this seems not too absurd: Let us adapt the fragment shader and adjust the `location` of the input variable there, too.
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	layout (location=1) in vec4 data_from_the_vertexshader;

	void main(){
		theColour= data_from_the_vertexshader;
	}
```

Now it's okay, again. 

How about 
```glsl
#version 450

layout (location=0) out vec4 theColour;

layout (location=5) in vec4 data_from_the_vertexshader;

void main(){
	theColour= data_from_the_vertexshader;
}
```
together with this?
```glsl
#version 450

layout (location=5) out vec4 colourdata_for_the_fragmentshader;

void main() {
    gl_PointSize=10.0;
    gl_Position = vec4(0.4,0.2,0.0,1.0);
    colourdata_for_the_fragmentshader=vec4(0.4,1.0,0.5,1.0);
}
```

Again: Works. Variable outputs from one shader are recognized by the next shader by their location. This location is given in form of a layout directive like `layout (location=5)` preceding the `in` or `out` keyword when declaring the variables in the shader program.

From the fragment shader
```glsl
	#version 450
	
	layout (location=0) out vec4 theColour;

	layout (location=0) in vec4 data_from_the_vertexshader;

	void main(){
		theColour= data_from_the_vertexshader;
	}
```
we also notice that the "in" and "out" variables have separate enumerations for these slots. Location 0 for in is not the same as location 0 for out.


Now the colour is not hardcoded in the fragment shader any more. It's hardcoded in the vertex shader. It's time to get some data to the vertex shader. 

[Continue](018_Vertexshader.md)
