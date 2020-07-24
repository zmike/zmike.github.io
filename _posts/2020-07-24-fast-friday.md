---
published: true
---
## Spec Harder

At this point, longtime readers are well aware of the [SPIR-V spec](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html) since I reference it in most posts. This spec covers all the base operations and types used in shaders, which are then (occasionally) documented more fully in the full [Vulkan spec](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html).

There comes a time, however, when the core SPIR-V spec is not enough to maintain compatibility with GLSL instructions. In such cases, there are additional specifications which extend the base functionality.

* [GLSL 450](https://www.khronos.org/registry/spir-v/specs/unified1/GLSL.std.450.html) provides many additional ops, primarily for calculation-related functionality
* Other specs

Perhaps now you see where I'm going with this.

Yes, any time you're working with one of the "other specs", e..g, `SPV_KHR_vulkan_memory_model`, there's no longer this nice, consistent webpage for viewing the spec. Instead, the SPIR-V [landing page](https://www.khronos.org/registry/spir-v/) directs readers to this [GitHub repository](https://github.com/KhronosGroup/SPIRV-Registry), which then has a list of links to all the extensions, whereupon finally the goal is reached at a [meta-link](http://htmlpreview.github.io/?https://github.com/KhronosGroup/SPIRV-Registry/blob/master/extensions/KHR/SPV_KHR_vulkan_memory_model.html) that displays the version in the repo.

The frustrating part about this is that you have to know that you need to do this. Upon seeing references to an extension in the SPIR-V spec, and they are most certainly listed, the next step is naturally to find more info on the extension. But there is no info on any extensions in the core spec, and, while the extension names look like they're links, they aren't links at all, and so there's no way to directly get to the spec for an extension that you now know you need to use after seeing it as a requirement in the core spec.

The next step is a staple of any workflow. Typically, it's enough to just throw down a `search engine of your choice` query for "function name" or "specification name" or "extension name" when working with GL or VK, but for this one odd case, assuming your search engine even finds results, most likely you'll end up instead at [this page for SPV_KHR_vulkan_memory_model](https://github.com/KhronosGroup/SPIRV-Registry/blob/master/extensions/KHR/SPV_KHR_vulkan_memory_model.html), which isn't very useful.
