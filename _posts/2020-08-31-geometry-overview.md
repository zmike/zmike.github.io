---
published: true
---
## At Last, Geometry Shaders

I've mentioned GS on a few occasions in the past without going too deeply into the topic. The truth is that I'm probably not ever going to get that deep into the topic.

There's just not that much to talk about.

## The 101
Geometry shaders take the vertices from previous shader stages (vertex, tessellation) and then emit some sort of (possibly-complete) primitive. `EmitVertex()` and `EndPrimitive()` are the big GLSL functions that need to be cared about, and there's `gl_PrimitiveID` and `gl_InvocationID`, but hooking everything into a mesa (gallium) driver is, at the overview level, pretty straightforward:
* add `struct pipe_context` hooks for the gs state creation/deletion, which are just the same wrappers that every other shader type uses
* make sure to add GS state saving to any `util_blitter` usage
* add a bunch of GS shader pipe cap handling
* the driver now supports geometry shaders

## But Then Zink
The additional changes needed in zink are almost entirely in `ntv`. Most of this is just expanding existing conditionals which were previously restricted to vertex shaders to continue handling the input/output variables in the same way, though there's also just a bunch of boilerplate SPIRV setup for enabling/setting execution modes and the like that can mostly be handled with ctrl+f in the SPIRV spec with `shader_info.h` open.

One small gotcha is again `gl_Position` handling. Since this was previously transformed in the vertex shader to go from an OpenGL `[-1, 1]` depth range to a Vulkan `[0, 1]` range, any time a GS loads `gl_Position` it then needs to be un-transformed, as (at the point of the GS implementation patch) zink doesn't support shader keys, and so there will only ever be one variant of a shader, which means the vertex shader must always perform this transform. This will be resolved at a later point, but it's worth taking note of now.

The other point to note is, as always, I know everyone's tired of it by now, **transform feedback**. Whereas vertex shaders emit this info at the end of the shader, things are different in the case of geometry shaders. The point of transform feedback is to emit captured data at the time shader output occurs, so now xfb happens at the point of `EmitVertex()`.

And that's it.

Sometimes, but only sometimes, things in zink-land don't devolve into insane hacks.
