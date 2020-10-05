---
published: false
---
## Healthier Blogging

I took some time off to focus on making the numbers go up, but if I only do that sort of junk food style blogging with images, and charts, and *benchmarks* then we might all stop learning things, and certainly I won't be transferring any knowledge between the coding part of my brain and the speaking part, so really we're all losers at that point.

In other words, let's get back to going extra deep into some code.

## Descriptor Management
First: what are descriptors?

Descriptors are, in short, when you feed a buffer or an image (+sampler) into a shader. In OpenGL, this is all handled for the user behind the scenes with e.g., a simple `glGenBuffers()` -> `glBindBuffer()` -> `glBufferData()` for an attached buffer. For a gallium-based driver, this example case will trigger the `pipe_context::set_shader_buffers` hook at draw time to inform the driver that a ssbo has been attached, and then the driver can link it up with the GPU.

Things are a bit different in Vulkan. There's [an entire chapter](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#descriptorsets) of the spec devoted to explaining how descriptors work in great detail, but the important details for the zink case are:
* each descriptor must have created for it a `binding` value which is unique for the given descriptor set
* each descriptor must have a Vulkan descriptor type, as converted from its OpenGL type
* each descriptor must also potentially expand to its full array size (image descriptor types only)

Additionally, while organizing and initializing all these descriptor sets, zink has to track all the resources used and guarantee their lifetimes exceed the lifetimes of the batch they're being submitted with.

To handle this, zink has *an amount* of code. In the current state of the repo, it's [about 40 lines](https://gitlab.freedesktop.org/mesa/mesa/-/blob/08d51e92aee0cddc5ad567dddd432cc4016a4570/src/gallium/drivers/zink/zink_draw.c#L293).

However...

[![meme-update_descriptors.png]({{site.url}}/assets/meme-update_descriptors.png)]({{site.url}}/assets/meme-update_descriptors.png)

In the state of my branch that I'm going to be working off of for the next few blog posts, the function for handling descriptor updates is [304 lines](https://gitlab.freedesktop.org/zmike/mesa/-/blob/c1fc05e5e48b5a259e3d53cbaf002505047933b6/src/gallium/drivers/zink/zink_draw.c#L279).