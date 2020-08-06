---
published: false
---
## Post-Storm Posting

I'm back, and today's topic is testing.

Again.

But this time is different. This time I'm going to be looking into a specific test format, namely piglit shader tests.

Shader tests in piglit are tests which are passed through the `shader_runner` binary, which parses `.shader_test` files to automatically generate tests based on GLSL without requiring any actual GL code. This makes writing tests easy, and, more importantly for my own use case, it makes debugging them easier.

## An Example
```
[require]
GLSL >= 1.50

[vertex shader passthrough]

[fragment shader]
#version 150

uniform float c;

out vec4 color;

void main()
{
	// set the 'green' value of the color to the value of the uniform
	color = vec4(0.0, c, 0.0, 1.0);
}

[test]
clear color 0.2 0.2 0.2 0.2
clear

// set 'c' to 1
uniform float c 1;
// draw a rect starting at -1,-1 (bottom left) with a width of 2 units, covering the fb
draw rect -1 -1 2 2
// read back the color buffer at [0.0, 0.0] (bottom left) and compare against rgb(0.0, 1.0, 0.0)
relative probe rect rgb (0.0, 0.0, 0.5, 0.5) (0.0, 1.0, 0.0)
```
Here's an extremely simple example with a passthrough vertex shader (i.e., `gl_Position = piglit_vertex;`) which draws a full-fb rectangle based on the color passed in as a uniform value and then reads back the color in the middle of the fb to check that it's green.

## Debugging
The great part of shader tests like these is that they're very simple to debug in a visual way. For example, if the above test was failing, I'd have a number of easy ways to narrow down where the problem was:
* replace `c` in the output color with a constant value to see if the color is being drawn correctly
* change the first two components of the `probe` command's coordinates to see if perhaps the failure is from the read failing in the specified location in the color buffer
* change the size of the rectangle being drawn in the event that it isn't as large as it should be

And through all of this, no rebuilding is necessary; I can just make my edits and then `shader_runner` will generate the test at runtime.

## More Complicated
Yes, they can get a bit more complex as well. For example, here's one that I was using while working on dynamic UBO indexing:
```
# This test verifies that dynamically uniform indexing of sampler arrays
# in the fragment shader behaves correctly.

[require]
GLSL >= 1.50
GL_ARB_gpu_shader5

[vertex shader passthrough]

[fragment shader]
#version 150
#extension GL_ARB_gpu_shader5: require

uniform sampler2D s[4];

uniform int n;

out vec4 color;

void main()
{
	color = texture(s[n], vec2(0.75, 0.25));
}

[test]
clear color 0.2 0.2 0.2 0.2
clear

uniform int s[0] 0
uniform int s[1] 1
uniform int s[2] 2
uniform int s[3] 3

texture checkerboard 0 0 (32, 32) (0.5, 0.0, 0.0, 0.0) (1.0, 0.0, 0.0, 0.0)
texparameter 2D min nearest
texparameter 2D mag nearest

texture checkerboard 1 0 (32, 32) (0.5, 0.0, 0.0, 0.0) (0.0, 1.0, 0.0, 0.0)
texparameter 2D min nearest
texparameter 2D mag nearest

texture checkerboard 2 0 (32, 32) (0.5, 0.0, 0.0, 0.0) (0.0, 0.0, 1.0, 0.0)
texparameter 2D min nearest
texparameter 2D mag nearest

texture checkerboard 3 0 (32, 32) (0.5, 0.0, 0.0, 0.0) (1.0, 1.0, 1.0, 1.0)
texparameter 2D min nearest
texparameter 2D mag nearest

uniform int n 0
draw rect -1 -1 1 1

relative probe rect rgb (0.0, 0.0, 0.5, 0.5) (1.0, 0.0, 0.0)

uniform int n 1
draw rect 0 -1 1 1

relative probe rect rgb (0.5, 0.0, 0.5, 0.5) (0.0, 1.0, 0.0)

uniform int n 2
draw rect -1 0 1 1

relative probe rect rgb (0.0, 0.5, 0.5, 0.5) (0.0, 0.0, 1.0)

uniform int n 3
draw rect 0 0 1 1

relative probe rect rgb (0.5, 0.5, 0.5, 0.5) (1.0, 1.0, 1.0)
```
This test again uses a passthrough vertex shader, outputting the color from one of the members of the sampler uniform array using another uniform as the array index. The texture being sampled in each case goes through a utility function `piglit_checkerboard_texture()`, which takes parameters:
 * **tex**                Name of the texture to be used.
 * **level**              Mipmap level the checkerboard should be written to
 * **width**              Width of the texture image
 * **height**             Height of the texture image
 * **horiz_square_size**  Size of each checkerboard tile along the X axis
 * **vert_square_size**   Size of each checkerboard tile along the Y axis
 * **black**              RGBA color to be used for "black" tiles
 * **white**              RGBA color to be used for "white" tiles
 
 Each `draw` command then draws a rectangle in a different corner of the framebuffer, sampling from a texture with a different color, which produces:
![arb_gpu_shader5-sampler_array_indexing-fs-simple.png]({{site.url}}/assets/arb_gpu_shader5-sampler_array_indexing-fs-simple.png)

While I was getting this to work, I had a couple options available to me so that I could break down the test and run it in smaller pieces:
* eliminate all but the first texture, draw, and probe calls for simpler shader output
* use a constant array index to verify that arrays of samplers work