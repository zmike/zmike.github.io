---
published: true
---
## Last of the Vertex Processors

As I've touched upon previously, zink does some work in `ntv` to perform translations of certain GLSL variables to SPIR-V versions of those variables. Also sometimes we add and remove variables for compatibility reasons.

The key part of these translations, for some cases, is to ensure that they occur on the right shader. As an example that I've talked about several times, Vulkan coordinates are [different](https://matthewwellings.com/blog/the-new-vulkan-coordinate-system/) from GL coordinates, specifically in that the Z axis is compressed and thus values there must be converted to perform as expected in the underlying Vulkan driver. This means that `gl_Position` needs to be converted before reaching the fragment shader.

## Conversion Timing
Early approaches to handling this in zink, as in the currently-released versions of mesa, performed all translation in the vertex shader. Only vertex and fragment shaders are supported here, so this is fine and makes total sense.

Once more types of shaders become supported, however, this is no longer quite as good to be doing. Consider the following progression:
* vertex shader converts `gl_Position`
* vertex shader outputs `gl_Position`
* geometry shader consumes `gl_Position`
* geometry shader uses `gl_Position` for vertex output
* geometry shader outputs `gl_Position`

In this scenario, the geometry shader is still executing instructions that assume a GL coordinate input, which means they will not function as expected when they receive the converted VK coordinates. The simplest fix results in:
* vertex shader converts `gl_Position`
* vertex shader outputs `gl_Position`
* geometry shader consumes `gl_Position`
* geometry shader unconverts `gl_Position`
* geometry shader uses `gl_Position` for vertex output
* geometry shader converts `gl_Position`
* geometry shader outputs `gl_POsition`

I say simplest here because this requires no changes to the shader compiler ordering in zink, meaning that shaders don't need to be "aware" of each other, e.g., a vertex shader doesn't need to know whether a geometry shader exists and can just do the same conversion in every case. This is useful because at present, shaders in zink are compiled in a "random" order, meaning that it's impossible to know whether a geometry shader exists at the time that a vertex shader is being compiled.

This is still not ideal, however, as it means that the vertex and geometry shaders are going to be executing unnecessary instructions, which yields a big frowny face in benchmarks (probably not actually that big, but this is the sort of optimizing that lets you call your code "lightweight"). The situation is further complicated with the introduction of tessellation shaders, where the flow now starts looking like:
* vertex shader converts `gl_Position`
* vertex shader outputs `gl_Position`
* tessellation shader consumes `gl_Position`
* tessellation shader unconverts `gl_Position`
* tessellation shader converts `gl_Position`
* tessellation shader outputs `gl_Position`
* geometry shader consumes `gl_Position`
* geometry shader unconverts `gl_Position`
* geometry shader uses `gl_Position` for vertex output
* geometry shader converts `gl_Position`
* geometry shader outputs `gl_Position`

Not great.

## Once Per Pipeline
The obvious change required here is to ensure that zink [compiles shaders in pipeline order](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5970). With this done, all the uncompiled shaders are available to scan for info and existence, and the process can now be:
* vertex shader converts `gl_Position` **if no tessellation or geometry shader is present**
* vertex shader outputs `gl_Position`
* tessellation shader consumes `gl_Position`
* tessellation shader converts `gl_Position` **if no geometry shader is present**
* tessellation shader outputs `gl_Position`
* geometry shader consumes `gl_Position`
* geometry shader uses `gl_Position` for vertex output
* geometry shader converts `gl_Position`
* geometry shader outputs `gl_Position`

Now that's some *lightweight* shader execution. #optimized
