---
published: false
---
## Deeper Into Software

I don't feel like blogging about zink today, so here's more about everyone's favorite software implementation of Vulkan.

The existing LLVMpipe architecture works like this from a top-down view:
* `mesa / st` - this is the GL/Gallium state tracker
* `llvmpipe` - this is the Gallium driver
* `gallivm` - this is the LLVM program compiler
* `llvm` - this is where the fragment shader runs

In short, everything is for the purpose of compiling LLVM programs which will draw/compute the desired result.

Lavapipe makes a slight change:
* `lavapipe` - this is the Vulkan state tracker
* `llvmpipe` - this is the Gallium driver
* `gallivm` - this is the LLVM program compiler
* `llvm` - this is where the fragment shader runs

It's that simple.

Thus, any time a new feature is added to Lavapipe, what's actually being done is plumbing that Vulkan feature through some number of layers to change how LLVM is executed. Some features, like `samplerAnisotropy`, require significant work at the `gallivm` layer just to toggle a boolean flag at the `lavapipe` level.

Other changes, like [KHR_timeline_semaphores](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_KHR_timeline_semaphore.html) are entirely contained in Lavapipe.