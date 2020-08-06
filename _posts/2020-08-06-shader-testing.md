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