---
published: true
---
## Ecosystem Victory

Today marks (at last) the [release](https://github.com/KhronosGroup/Vulkan-Docs/commit/45af5eb1f66898c9f382edc5afd691aeb32c10c0) of some cool extensions I've had the pleasure of working on:
* [VK_EXT_graphics_pipeline_library](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VK_EXT_graphics_pipeline_library.html)
* [VK_EXT_primitives_generated_query](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VK_EXT_primitives_generated_query.html)

## VK_EXT_graphics_pipeline_library
This extension revolutionizes how PSOs can be managed by the application, and it's the first step towards solving the dreaded stuttering that zink suffers from when attempting to play any sort of game. There's definitely going to be more posts from me on this in the future.

## VK_EXT_primitives_generated_query
Currently, zink has to do some awfulness internally to replicate the awfulness of `GL_PRIMITIVES_GENERATED`. With this extension, at least some of that awfulness can be pushed down to the driver. And the spec, of course. You can't scrub this filth out of your soul.

## Support
The mesa community being awesome as it is, support for these extensions is already underway:
* ANV has merge requests up already for [both](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/15638) of [them](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/15637)
* RADV has a merge request up for preliminary support of [VK_EXT_primitives_generated_query](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/15639) on certain hardware, and VK_EXT_graphics_pipeline_library support is nearing completion

But obviously Lavapipe, being the greatest of all drivers, will [already have support landed](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/15636) by the time you read this post.

Let the bug reports flow!
