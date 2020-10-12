---
published: false
---
## Jumping Right In

When last I left off, I'd cleared out 2/3 of my checklist for improving `update_sampler_descriptors()` performance:
* ~~[add_transition()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L223), which is a function for accumulating and merging memory barriers for resources using a hash table
* ~~[bind_descriptors()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L272), which calls [vkUpdateDescriptorSets](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkUpdateDescriptorSets.html) and [vkCmdBindDescriptorSets](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBindDescriptorSets.html)~~
* [handle_image_descriptor()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L467), which is a helper for setting up sampler/image descriptors

`handle_image_descriptor()` was next on my list. Looking at the flamegraph again yielded the culprit:

[![frame_retrieval.png]({{site.url}}/assets/desc_profiling1/frame_retrieval.png)]({{site.url}}/assets/desc_profiling1/frame_retrieval.png)

It's not visible due to this being a screenshot, but the whole of the perf hog here was obvious, so let's check out the function itself since it's small. I think I explored part of this at one point in the distant past, possibly for `ARB_texture_buffer_object`, but refactoring has changed things up a bit:
```c
static void
handle_image_descriptor(struct zink_screen *screen, struct zink_resource *res, enum zink_descriptor_type type, VkDescriptorType vktype, VkWriteDescriptorSet *wd,
                        VkImageLayout layout, unsigned *num_image_info, VkDescriptorImageInfo *image_info, struct zink_sampler_state *sampler,
                        VkBufferView *null_view, VkImageView imageview, bool do_set)
```
First, yes, there's a lot of parameters. There's a lot of them, including `VkBufferView *null_view`, which is a pointer to a stack array that's initialized as containing `VK_NULL_HANDLE`. As this struct must be initialized with a pointer to an array, it's important that the stack variable used doesn't go out of scope, so it has to be passed in like this or else this functionality can't be broken out in this way.
```c
{
    if (!res) {
        /* if we're hitting this assert often, we can probably just throw a junk buffer in since
         * the results of this codepath are undefined in ARB_texture_buffer_object spec
         */
        assert(screen->info.rb2_feats.nullDescriptor);
        
        switch (vktype) {
        case VK_DESCRIPTOR_TYPE_UNIFORM_TEXEL_BUFFER:
        case VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER:
           wd->pTexelBufferView = null_view;
           break;
        case VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER:
        case VK_DESCRIPTOR_TYPE_STORAGE_IMAGE:
           image_info->imageLayout = VK_IMAGE_LAYOUT_UNDEFINED;
           image_info->imageView = VK_NULL_HANDLE;
           if (sampler)
              image_info->sampler = sampler->sampler[0];
           if (do_set)
              wd->pImageInfo = image_info;
           ++(*num_image_info);
           break;
        default:
           unreachable("unknown descriptor type");
        }
```
This is just handling for null shader inputs, which is permitted by various GL specs.
```c
     } else if (res->base.target != PIPE_BUFFER) {
        assert(layout != VK_IMAGE_LAYOUT_UNDEFINED);
        image_info->imageLayout = layout;
        image_info->imageView = imageview;
        if (sampler) {
           VkFormatProperties props;
           vkGetPhysicalDeviceFormatProperties(screen->pdev, res->format, &props);
```
This [vkGetPhysicalDeviceFormatProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkGetPhysicalDeviceFormatProperties.html) call is actually the entire cause of `handle_image_descriptor()` using any CPU time, at least on ANV. The lookup for the format is a significant bottleneck here, so it has to be removed.
```c
           if ((res->optimial_tiling && props.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT) ||
               (!res->optimial_tiling && props.linearTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT))
              image_info->sampler = sampler->sampler[0];
           else
              image_info->sampler = sampler->sampler[1] ?: sampler->sampler[0];
        }
        if (do_set)
           wd->pImageInfo = image_info;
        ++(*num_image_info);
     }
}
```
Just for completeness, the remainder of this function is checking whether the device's format features support the requested type of filtering (if `linear`), and then zink will fall back to `nearest` in other cases. Following this, `do_set` is true only for the base member of an image/sampler array of resources, and so this is the one that gets added into the descriptor set.

But now I'm again returning to `vkGetPhysicalDeviceFormatProperties`. Since this is using CPU, it needs to get out of the hotpath here in descriptor updating, but it does still need to be called. As such, I've 