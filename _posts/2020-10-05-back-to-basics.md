---
published: false
---
## Healthier Blogging

I took some time off to focus on making the numbers go up, but if I only do that sort of junk food style blogging with images, and charts, and *benchmarks* then we might all stop learning things, and certainly I won't be transferring any knowledge between the coding part of my brain and the speaking part, so really we're all losers at that point.

In other words, let's get back to going extra deep into some code.

## Descriptor Management
First: what are descriptors?

Descriptors are, in short, when you feed a buffer or an image (+sampler) into a shader. In OpenGL, this is all handled for the user behind the scenes with e.g., a simple `glGenBuffers()` -> `glBindBuffer()` -> `glBufferData()` for an attached buffer. For a gallium-based driver, this example case will trigger the `pipe_context::set_shader_buffers` hook at draw time to inform the driver that a ssbo has been attached, and then the driver can link it up with the GPU.

Things are a bit different in Vulkan. There's [an entire chapter](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#descriptorsets) of the spec devoted to explaining how descriptors work in great detail