---
published: false
---
## New Week, New Idea

I have to change things up.

Historically I've spent a day working on zink and then written up a post at the end. The problem with this approach is obvious: a lot of times when I get to the end of the day I'm just too mentally drained to think about anything else and want to just pass out on my couch.

So now I'm going to try inverting my schedule: as soon as I get up, it's now going to be blog time.

I'm not even fully awake right now, so this is definitely going to be interesting.

## Stencil Sampling
Today's exploratory morning post is about sampling from stencil buffers.

_What is sampling from stencil buffers_, some of you might be asking.

Sampling in general is the reading of data from a resource. It's most commonly used as an alternative to using a **Copy** command for transferring some amount of data from one resource to another in a specified way.

For example, extracting only stencil data from a resource which combines both depth and stencil data. In zink, this is an important operation because none of the *Copy* commands support multisampled resources containing both depth and stencil data, an OpenGL feature that the unit tests most certainly cover.

As with all things, zink has a tough time with this.

## Sampling Basics
For the purpose of this post, I'm only going to be talking about sampling from image resources. Sampling from buffer resources is certainly possible and useful, however, but there's just less that can go wrong for that case.

The general process of a sampling-based copy operation in Gallium-based drivers is as follows:
* have resource _src_ which contains some amount of data
* have resource _dst_ which is a the intended destination for the data from _src_
* bind _src_ as a "sampler view", which is essentially a combination of info that determines how data will be sampled from a resource
* bind _dst_ as an output target (e.g., a framebuffer attachment)
* bind a fragment shader containing a [sampler type](https://www.khronos.org/opengl/wiki/Sampler_(GLSL)) that samples from the bound sampler view and writes to the output target (either by `gl_FragColor` output or `imageStore`)
* dump some vertices into the pipeline and blammo, missionaccomplished.jpg

In the case of stencil sampling, zink has issues with the third step here.

## The Code
Here's what we've currently got shipping in the driver for the relevant part of creating image sampler views:

```c
VkImageViewCreateInfo ivci = {};
ivci.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
ivci.image = res->image;
ivci.viewType = image_view_type(state->target);
ivci.format = zink_get_format(screen, state->format);
assert(ivci.format);
ivci.components.r = component_mapping(state->swizzle_r);
ivci.components.g = component_mapping(state->swizzle_g);
ivci.components.b = component_mapping(state->swizzle_b);
ivci.components.a = component_mapping(state->swizzle_a);

ivci.subresourceRange.aspectMask = sampler_aspect_from_format(state->format);
ivci.subresourceRange.baseMipLevel = state->u.tex.first_level;
ivci.subresourceRange.baseArrayLayer = state->u.tex.first_layer;
ivci.subresourceRange.levelCount = state->u.tex.last_level - state->u.tex.first_level + 1;
ivci.subresourceRange.layerCount = state->u.tex.last_layer - state->u.tex.first_layer + 1;

err = vkCreateImageView(screen->dev, &ivci, NULL, &sampler_view->image_view);
```
Filling in some gaps:
* `res` is the image being sampled from
* `image_view_type()` converts Gallium texture types (e.g., 1D, 2D, 2D_ARRAY, ...) to the corresponding Vulkan type
* `zink_get_format()` converts a Gallium image format to a usage Vulkan one
* `component_mapping()` converts a Gallium swizzle to a Vulkan one (swizzles determine which channel in the sample operation are mapped from the source to the destination)
* `sampler_aspect_from_format()` infers `VkImageAspectFlags` from a Gallium format

## The Problem
Regarding sampler descriptors, Vulkan spec states [If imageView is created from a depth/stencil image, the aspectMask used to create the imageView must include either VK_IMAGE_ASPECT_DEPTH_BIT or VK_IMAGE_ASPECT_STENCIL_BIT but not both](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#VUID-VkDescriptorImageInfo-imageView-01976).

This means that for combined depth+stencil resources, only the depth or stencil aspect can be specified but not both. As Gallium presents drivers with a format and swizzle based on the data being sampled from the image's data, this poses a problem since 1) the format provided will usually map to something like `VK_FORMAT_D32_SFLOAT_S8_UINT` and 2) the swizzle provided will be based on this format.

But if zink can only specify one of the aspects, this poses a problem.

## The Solution
The format being sampled must also match the aspect type, and `VK_FORMAT_D32_SFLOAT_S8_UINT` is obviously not a pure stencil format. This means that any time zink infers a stencil-only aspect image format like `PIPE_FORMAT_X32_S8X24_UINT`, which is a two channel format where the depth channel is ignored, the format passed in `VkImageViewCreateInfo` has to just be the stencil format being sampled. Helpfully, this will always be `VK_FORMAT_S8_UINT`.

So now the code would look like this:

```c
VkImageViewCreateInfo ivci = {};
ivci.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
ivci.image = res->obj->image;
ivci.viewType = image_view_type(state->target);

ivci.components.r = component_mapping(state->swizzle_r);
ivci.components.g = component_mapping(state->swizzle_g);
ivci.components.b = component_mapping(state->swizzle_b);
ivci.components.a = component_mapping(state->swizzle_a);
ivci.subresourceRange.aspectMask = sampler_aspect_from_format(state->format);
/* samplers for stencil aspects of packed formats need to always use stencil type */
if (ivci.subresourceRange.aspectMask == VK_IMAGE_ASPECT_STENCIL_BIT)
   ivci.format = VK_FORMAT_S8_UINT;
else
   ivci.format = zink_get_format(screen, state->format);
```

## Mo' Time, Mo' Problems
The above code was working great for months in zink-wip.

Then "bugs" were fixed in master.

The new problem came from a [merge request](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7922) claiming to "[fix depth/stencil blit shaders](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7922/diffs?commit_id=7ca72f172678116d29d254b786a9422b864aef3d)". The short of this is that previously, the shaders generated by mesa for the purpose of doing depth+stencil sampling were always reading from the **first channel** of the image, which was exactly what zink was intending given that for that case the underlying Vulkan driver would only be reading one component anyway. After this change, however, samplers are now reading from the **second channel** of the image.

Given that a Vulkan stencil format has no second channel, this poses a problem.

Luckily, the magic of swizzles can solve this. By mapping the second channel of the sampler to the first channel of the image data, the sampler will read the stencil data again.

The fully fixed code now looks like this:
```c
VkImageViewCreateInfo ivci = {};
ivci.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
ivci.image = res->obj->image;
ivci.viewType = image_view_type(state->target);

ivci.components.r = component_mapping(state->swizzle_r);
ivci.components.g = component_mapping(state->swizzle_g);
ivci.components.b = component_mapping(state->swizzle_b);
ivci.components.a = component_mapping(state->swizzle_a);
ivci.subresourceRange.aspectMask = sampler_aspect_from_format(state->format);
/* samplers for stencil aspects of packed formats need to always use stencil type */
if (ivci.subresourceRange.aspectMask == VK_IMAGE_ASPECT_STENCIL_BIT) {
   ivci.format = VK_FORMAT_S8_UINT;
   ivci.components.g = VK_COMPONENT_SWIZZLE_R;
} else
   ivci.format = zink_get_format(screen, state->format);
```

And now everything works.

For now.