---
published: false
---
## This is a title

I'm back.

Where did I go?

My birthday passed recently, so I gifted myself a couple weeks off from blogging. Feels good.

For today, this is a Lavapipe blog.

## What's New With Lavapipe?
Lots.

Let's check out what **conformant** features were added just in July:
* EXT_line_rasterization
* EXT_vertex_input_dynamic_state
* EXT_extended_dynamic_state2
* EXT_color_write_enable
* features.strictLines
* features.shaderStorageImageExtendedFormats
* features.shaderStorageImageReadWithoutFormat
* features.samplerAnisotropy
* KHR_timeline_semaphores

Also under the hood now is a [new 2D rasterizer from VMWare](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11969) which yields "a 2x to 3x performance improvement for 2D workloads".

## Why Aren't You Using Lavapipe Yet?
Have a big Vulkan-using project? Do you constantly have to worry about breakages from all manner of patches being merged without testing? Can't afford or too lazy to set up and maintain actual hardware for testing?

Why not Lavapipe?

Seriously, why not? If there's features missing that you need for your project, open tickets so we know what to work on.