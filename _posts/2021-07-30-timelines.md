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

## What Are Timeline Semaphores?
Vulkan has a number of mechanisms for synchronization including fences, events, and binary semaphores, all of which serve a specific purpose. For more conrete on all of them, please read [the blog of an actual expert](https://themaister.net/blog/2019/08/14/yet-another-blog-explaining-vulkan-synchronization/).

The best and most awesome (don't @ me, it's not debatable) of these synchronization methods, however is the timeline semaphore.

A timeline semaphore is an object that can be used to signal and wait on specific integer-assigned points in command execution, also known as *timelines*. Each queue submission can be accompanied by an array of timeline semaphores to wait on and an array to signal; command buffers in a given submission will wait before executing, then signal after they're done. This enables parallel code design where one thread can assemble command buffers and submit them, and the GPU can be made to pause at certain points for buffers/images referenced to become populated by another thread before continuing with execution.

Typically, semaphores are managed through signals which pass through the kernel and hardware, meaning that "waiting" on a timeline is really just waiting on an ioctl (DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT) to signal that the specified timeline id has occurred, which requires no additional host-side synchronization. Things get a bit trickier in software, however, as the kernel is not involved, so everything must be managed in the driver.

