---
published: false
---
## Jumping Right In

When last I left off, I'd cleared out 2/3 of my checklist for improving `update_sampler_descriptors()` performance:
* ~~[add_transition()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L223), which is a function for accumulating and merging memory barriers for resources using a hash table
* ~~[bind_descriptors()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L272), which calls [vkUpdateDescriptorSets](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkUpdateDescriptorSets.html) and [vkCmdBindDescriptorSets](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBindDescriptorSets.html)~~
* [handle_image_descriptor()](https://gitlab.freedesktop.org/zmike/mesa/-/blob/blog-20201008/src/gallium/drivers/zink/zink_draw.c#L467), which is a helper for setting up sampler/image descriptors

`handle_image_descriptor()` was next on my list. As a result, I immediately dove right into an entirely different part of the flamegraph since I'd just been struck by a seemingly-obvious idea. Here's the last graph:
[![frame_retrieval.png]({{site.url}}/assets/desc_profiling1/frame_retrieval.png)]({{site.url}}/assets/desc_profiling1/frame_retrieval.png)

Here's the next step:
[![reuse_barriers.png]({{site.url}}/assets/desc_profiling1/reuse_barriers.png)]({{site.url}}/assets/desc_profiling1/reuse_barriers.png)

What changed?

Well, as I'm now caching descriptor sets across descriptor pools, it occurred to me that, assuming my descriptor state hashing mechanism is accurate, all the resources used in a given set must be identical. This means that all resources for a given type (e.g., UBO, SSBO, sampler, image) must be completely identical to previous uses across all shader stages. Extrapolating further, this *also* means that the way in which these resources are used must also identical, which means the pipeline barriers for access and image layouts must also be identical.

Which means they can be stored onto the `struct zink_descriptor_set` object and reused instead of being accumulated every time. This reuse completely eliminates `add_transition()` from using any CPU time (it's the left-most block above `update_sampler_descriptors()` in the first graph), and it thus massively reduces overall time for descriptor updates.

This marks a notable landmark, as it's the point at which `update_descriptors()` begins to use only ~50% of the total CPU time consumed in `zink_draw_vbo()`, with the other half going to the draw command where it should be.

## `handle_image_descriptor()`
At last my optimization-minded workflow returned to this function, and looking at the flamegraph again yielded the culprit. It's not visible due to this being a screenshot, but the whole of the perf hog here was obvious, so let's check out the function itself since it's small. I think I explored part of this at one point in the distant past, possibly for `ARB_texture_buffer_object`, but refactoring has changed things up a bit:
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

But now I'm again returning to `vkGetPhysicalDeviceFormatProperties`. Since this is using CPU, it needs to get out of the hotpath here in descriptor updating, but it does still need to be called. As such, I've thrown more of this onto the `zink_screen` object:
```c
static void
populate_format_props(struct zink_screen *screen)
{
   for (unsigned i = 0; i < PIPE_FORMAT_COUNT; i++) {
      VkFormat format = zink_get_format(screen, i);
      if (!format)
         continue;
      vkGetPhysicalDeviceFormatProperties(screen->pdev, format, &screen->format_props[i]);
   }
}
```
Indeed, now instead of performing the fetch on every decsriptor update, I'm just grabbing all the properties on driver init and then using the cached values throughout. Let's see how this looks.
[![props.png]({{site.url}}/assets/desc_profiling1/props.png)]({{site.url}}/assets/desc_profiling1/props.png)

`update_descriptors()` is now using visibly less time than the draw command, though not by a huge amount. I'm also now up to an unstable **33fps**.

## Some Cleanups
At this point, it bears mentioning that I wasn't entirely satisfied with the amount of CPU consumed by descriptor state hashing, so I ended up doing some pre-hashing here for samplers, as they have the largest state. At the beginning, the hashing looked like this:
[![](https://mermaid.ink/img/eyJjb2RlIjoic3RhdGVEaWFncmFtXG5bKl0gLS0-IHMxXG5zMSAtLT4gczJcbnMyIC0tPiBzM1xuczMgLS0-IHM0XG5zNCAtLT4gczVcbnN0YXRlIFwiY3JlYXRlIHNhbXBsZXIgYW5kIHNhbXBsZXIgdmlld1wiIGFzIHMxXG5zdGF0ZSBcInBvc3NpYmx5IGludmFsaWRhdGUgc2FtcGxlciBkZXNjcmlwdG9yIHN0YXRlc1wiIGFzIHMyXG5zdGF0ZSBcInppbmtfZHJhd192Ym9cIiBhcyBzM1xuc3RhdGUgXCJ1cGRhdGVfZGVzY3JpcHRvcnNcIiBhcyBzNFxuc3RhdGUgXCJyZWhhc2ggc2FtcGxlciBzdGF0ZXNcIiBhcyBzNVxuIiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQiLCJ0aGVtZVZhcmlhYmxlcyI6eyJiYWNrZ3JvdW5kIjoid2hpdGUiLCJwcmltYXJ5Q29sb3IiOiIjRUNFQ0ZGIiwic2Vjb25kYXJ5Q29sb3IiOiIjZmZmZmRlIiwidGVydGlhcnlDb2xvciI6ImhzbCg4MCwgMTAwJSwgOTYuMjc0NTA5ODAzOSUpIiwicHJpbWFyeUJvcmRlckNvbG9yIjoiaHNsKDI0MCwgNjAlLCA4Ni4yNzQ1MDk4MDM5JSkiLCJzZWNvbmRhcnlCb3JkZXJDb2xvciI6ImhzbCg2MCwgNjAlLCA4My41Mjk0MTE3NjQ3JSkiLCJ0ZXJ0aWFyeUJvcmRlckNvbG9yIjoiaHNsKDgwLCA2MCUsIDg2LjI3NDUwOTgwMzklKSIsInByaW1hcnlUZXh0Q29sb3IiOiIjMTMxMzAwIiwic2Vjb25kYXJ5VGV4dENvbG9yIjoiIzAwMDAyMSIsInRlcnRpYXJ5VGV4dENvbG9yIjoicmdiKDkuNTAwMDAwMDAwMSwgOS41MDAwMDAwMDAxLCA5LjUwMDAwMDAwMDEpIiwibGluZUNvbG9yIjoiIzMzMzMzMyIsInRleHRDb2xvciI6IiMzMzMiLCJtYWluQmtnIjoiI0VDRUNGRiIsInNlY29uZEJrZyI6IiNmZmZmZGUiLCJib3JkZXIxIjoiIzkzNzBEQiIsImJvcmRlcjIiOiIjYWFhYTMzIiwiYXJyb3doZWFkQ29sb3IiOiIjMzMzMzMzIiwiZm9udEZhbWlseSI6IlwidHJlYnVjaGV0IG1zXCIsIHZlcmRhbmEsIGFyaWFsIiwiZm9udFNpemUiOiIxNnB4IiwibGFiZWxCYWNrZ3JvdW5kIjoiI2U4ZThlOCIsIm5vZGVCa2ciOiIjRUNFQ0ZGIiwibm9kZUJvcmRlciI6IiM5MzcwREIiLCJjbHVzdGVyQmtnIjoiI2ZmZmZkZSIsImNsdXN0ZXJCb3JkZXIiOiIjYWFhYTMzIiwiZGVmYXVsdExpbmtDb2xvciI6IiMzMzMzMzMiLCJ0aXRsZUNvbG9yIjoiIzMzMyIsImVkZ2VMYWJlbEJhY2tncm91bmQiOiIjZThlOGU4IiwiYWN0b3JCb3JkZXIiOiJoc2woMjU5LjYyNjE2ODIyNDMsIDU5Ljc3NjUzNjMxMjglLCA4Ny45MDE5NjA3ODQzJSkiLCJhY3RvckJrZyI6IiNFQ0VDRkYiLCJhY3RvclRleHRDb2xvciI6ImJsYWNrIiwiYWN0b3JMaW5lQ29sb3IiOiJncmV5Iiwic2lnbmFsQ29sb3IiOiIjMzMzIiwic2lnbmFsVGV4dENvbG9yIjoiIzMzMyIsImxhYmVsQm94QmtnQ29sb3IiOiIjRUNFQ0ZGIiwibGFiZWxCb3hCb3JkZXJDb2xvciI6ImhzbCgyNTkuNjI2MTY4MjI0MywgNTkuNzc2NTM2MzEyOCUsIDg3LjkwMTk2MDc4NDMlKSIsImxhYmVsVGV4dENvbG9yIjoiYmxhY2siLCJsb29wVGV4dENvbG9yIjoiYmxhY2siLCJub3RlQm9yZGVyQ29sb3IiOiIjYWFhYTMzIiwibm90ZUJrZ0NvbG9yIjoiI2ZmZjVhZCIsIm5vdGVUZXh0Q29sb3IiOiJibGFjayIsImFjdGl2YXRpb25Cb3JkZXJDb2xvciI6IiM2NjYiLCJhY3RpdmF0aW9uQmtnQ29sb3IiOiIjZjRmNGY0Iiwic2VxdWVuY2VOdW1iZXJDb2xvciI6IndoaXRlIiwic2VjdGlvbkJrZ0NvbG9yIjoicmdiYSgxMDIsIDEwMiwgMjU1LCAwLjQ5KSIsImFsdFNlY3Rpb25Ca2dDb2xvciI6IndoaXRlIiwic2VjdGlvbkJrZ0NvbG9yMiI6IiNmZmY0MDAiLCJ0YXNrQm9yZGVyQ29sb3IiOiIjNTM0ZmJjIiwidGFza0JrZ0NvbG9yIjoiIzhhOTBkZCIsInRhc2tUZXh0TGlnaHRDb2xvciI6IndoaXRlIiwidGFza1RleHRDb2xvciI6IndoaXRlIiwidGFza1RleHREYXJrQ29sb3IiOiJibGFjayIsInRhc2tUZXh0T3V0c2lkZUNvbG9yIjoiYmxhY2siLCJ0YXNrVGV4dENsaWNrYWJsZUNvbG9yIjoiIzAwMzE2MyIsImFjdGl2ZVRhc2tCb3JkZXJDb2xvciI6IiM1MzRmYmMiLCJhY3RpdmVUYXNrQmtnQ29sb3IiOiIjYmZjN2ZmIiwiZ3JpZENvbG9yIjoibGlnaHRncmV5IiwiZG9uZVRhc2tCa2dDb2xvciI6ImxpZ2h0Z3JleSIsImRvbmVUYXNrQm9yZGVyQ29sb3IiOiJncmV5IiwiY3JpdEJvcmRlckNvbG9yIjoiI2ZmODg4OCIsImNyaXRCa2dDb2xvciI6InJlZCIsInRvZGF5TGluZUNvbG9yIjoicmVkIiwibGFiZWxDb2xvciI6ImJsYWNrIiwiZXJyb3JCa2dDb2xvciI6IiM1NTIyMjIiLCJlcnJvclRleHRDb2xvciI6IiM1NTIyMjIiLCJjbGFzc1RleHQiOiIjMTMxMzAwIiwiZmlsbFR5cGUwIjoiI0VDRUNGRiIsImZpbGxUeXBlMSI6IiNmZmZmZGUiLCJmaWxsVHlwZTIiOiJoc2woMzA0LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTMiOiJoc2woMTI0LCAxMDAlLCA5My41Mjk0MTE3NjQ3JSkiLCJmaWxsVHlwZTQiOiJoc2woMTc2LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTUiOiJoc2woLTQsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSIsImZpbGxUeXBlNiI6ImhzbCg4LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTciOiJoc2woMTg4LCAxMDAlLCA5My41Mjk0MTE3NjQ3JSkifX0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic3RhdGVEaWFncmFtXG5bKl0gLS0-IHMxXG5zMSAtLT4gczJcbnMyIC0tPiBzM1xuczMgLS0-IHM0XG5zNCAtLT4gczVcbnN0YXRlIFwiY3JlYXRlIHNhbXBsZXIgYW5kIHNhbXBsZXIgdmlld1wiIGFzIHMxXG5zdGF0ZSBcInBvc3NpYmx5IGludmFsaWRhdGUgc2FtcGxlciBkZXNjcmlwdG9yIHN0YXRlc1wiIGFzIHMyXG5zdGF0ZSBcInppbmtfZHJhd192Ym9cIiBhcyBzM1xuc3RhdGUgXCJ1cGRhdGVfZGVzY3JpcHRvcnNcIiBhcyBzNFxuc3RhdGUgXCJyZWhhc2ggc2FtcGxlciBzdGF0ZXNcIiBhcyBzNVxuIiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQiLCJ0aGVtZVZhcmlhYmxlcyI6eyJiYWNrZ3JvdW5kIjoid2hpdGUiLCJwcmltYXJ5Q29sb3IiOiIjRUNFQ0ZGIiwic2Vjb25kYXJ5Q29sb3IiOiIjZmZmZmRlIiwidGVydGlhcnlDb2xvciI6ImhzbCg4MCwgMTAwJSwgOTYuMjc0NTA5ODAzOSUpIiwicHJpbWFyeUJvcmRlckNvbG9yIjoiaHNsKDI0MCwgNjAlLCA4Ni4yNzQ1MDk4MDM5JSkiLCJzZWNvbmRhcnlCb3JkZXJDb2xvciI6ImhzbCg2MCwgNjAlLCA4My41Mjk0MTE3NjQ3JSkiLCJ0ZXJ0aWFyeUJvcmRlckNvbG9yIjoiaHNsKDgwLCA2MCUsIDg2LjI3NDUwOTgwMzklKSIsInByaW1hcnlUZXh0Q29sb3IiOiIjMTMxMzAwIiwic2Vjb25kYXJ5VGV4dENvbG9yIjoiIzAwMDAyMSIsInRlcnRpYXJ5VGV4dENvbG9yIjoicmdiKDkuNTAwMDAwMDAwMSwgOS41MDAwMDAwMDAxLCA5LjUwMDAwMDAwMDEpIiwibGluZUNvbG9yIjoiIzMzMzMzMyIsInRleHRDb2xvciI6IiMzMzMiLCJtYWluQmtnIjoiI0VDRUNGRiIsInNlY29uZEJrZyI6IiNmZmZmZGUiLCJib3JkZXIxIjoiIzkzNzBEQiIsImJvcmRlcjIiOiIjYWFhYTMzIiwiYXJyb3doZWFkQ29sb3IiOiIjMzMzMzMzIiwiZm9udEZhbWlseSI6IlwidHJlYnVjaGV0IG1zXCIsIHZlcmRhbmEsIGFyaWFsIiwiZm9udFNpemUiOiIxNnB4IiwibGFiZWxCYWNrZ3JvdW5kIjoiI2U4ZThlOCIsIm5vZGVCa2ciOiIjRUNFQ0ZGIiwibm9kZUJvcmRlciI6IiM5MzcwREIiLCJjbHVzdGVyQmtnIjoiI2ZmZmZkZSIsImNsdXN0ZXJCb3JkZXIiOiIjYWFhYTMzIiwiZGVmYXVsdExpbmtDb2xvciI6IiMzMzMzMzMiLCJ0aXRsZUNvbG9yIjoiIzMzMyIsImVkZ2VMYWJlbEJhY2tncm91bmQiOiIjZThlOGU4IiwiYWN0b3JCb3JkZXIiOiJoc2woMjU5LjYyNjE2ODIyNDMsIDU5Ljc3NjUzNjMxMjglLCA4Ny45MDE5NjA3ODQzJSkiLCJhY3RvckJrZyI6IiNFQ0VDRkYiLCJhY3RvclRleHRDb2xvciI6ImJsYWNrIiwiYWN0b3JMaW5lQ29sb3IiOiJncmV5Iiwic2lnbmFsQ29sb3IiOiIjMzMzIiwic2lnbmFsVGV4dENvbG9yIjoiIzMzMyIsImxhYmVsQm94QmtnQ29sb3IiOiIjRUNFQ0ZGIiwibGFiZWxCb3hCb3JkZXJDb2xvciI6ImhzbCgyNTkuNjI2MTY4MjI0MywgNTkuNzc2NTM2MzEyOCUsIDg3LjkwMTk2MDc4NDMlKSIsImxhYmVsVGV4dENvbG9yIjoiYmxhY2siLCJsb29wVGV4dENvbG9yIjoiYmxhY2siLCJub3RlQm9yZGVyQ29sb3IiOiIjYWFhYTMzIiwibm90ZUJrZ0NvbG9yIjoiI2ZmZjVhZCIsIm5vdGVUZXh0Q29sb3IiOiJibGFjayIsImFjdGl2YXRpb25Cb3JkZXJDb2xvciI6IiM2NjYiLCJhY3RpdmF0aW9uQmtnQ29sb3IiOiIjZjRmNGY0Iiwic2VxdWVuY2VOdW1iZXJDb2xvciI6IndoaXRlIiwic2VjdGlvbkJrZ0NvbG9yIjoicmdiYSgxMDIsIDEwMiwgMjU1LCAwLjQ5KSIsImFsdFNlY3Rpb25Ca2dDb2xvciI6IndoaXRlIiwic2VjdGlvbkJrZ0NvbG9yMiI6IiNmZmY0MDAiLCJ0YXNrQm9yZGVyQ29sb3IiOiIjNTM0ZmJjIiwidGFza0JrZ0NvbG9yIjoiIzhhOTBkZCIsInRhc2tUZXh0TGlnaHRDb2xvciI6IndoaXRlIiwidGFza1RleHRDb2xvciI6IndoaXRlIiwidGFza1RleHREYXJrQ29sb3IiOiJibGFjayIsInRhc2tUZXh0T3V0c2lkZUNvbG9yIjoiYmxhY2siLCJ0YXNrVGV4dENsaWNrYWJsZUNvbG9yIjoiIzAwMzE2MyIsImFjdGl2ZVRhc2tCb3JkZXJDb2xvciI6IiM1MzRmYmMiLCJhY3RpdmVUYXNrQmtnQ29sb3IiOiIjYmZjN2ZmIiwiZ3JpZENvbG9yIjoibGlnaHRncmV5IiwiZG9uZVRhc2tCa2dDb2xvciI6ImxpZ2h0Z3JleSIsImRvbmVUYXNrQm9yZGVyQ29sb3IiOiJncmV5IiwiY3JpdEJvcmRlckNvbG9yIjoiI2ZmODg4OCIsImNyaXRCa2dDb2xvciI6InJlZCIsInRvZGF5TGluZUNvbG9yIjoicmVkIiwibGFiZWxDb2xvciI6ImJsYWNrIiwiZXJyb3JCa2dDb2xvciI6IiM1NTIyMjIiLCJlcnJvclRleHRDb2xvciI6IiM1NTIyMjIiLCJjbGFzc1RleHQiOiIjMTMxMzAwIiwiZmlsbFR5cGUwIjoiI0VDRUNGRiIsImZpbGxUeXBlMSI6IiNmZmZmZGUiLCJmaWxsVHlwZTIiOiJoc2woMzA0LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTMiOiJoc2woMTI0LCAxMDAlLCA5My41Mjk0MTE3NjQ3JSkiLCJmaWxsVHlwZTQiOiJoc2woMTc2LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTUiOiJoc2woLTQsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSIsImZpbGxUeXBlNiI6ImhzbCg4LCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJmaWxsVHlwZTciOiJoc2woMTg4LCAxMDAlLCA5My41Mjk0MTE3NjQ3JSkifX0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)

In this case, each sampler descriptor hash was the size of a [VkDescriptorImageInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorImageInfo.html), which is 2x 64bit values and a 32bit value, or 20bytes of hashing per sampler. That ends up being a lot of hashing, and it also ends up being a lot of repeatedly hashing the same values.

Instead, I changed things around to do some pre-hashing:
[![](https://mermaid.ink/img/eyJjb2RlIjoic3RhdGVEaWFncmFtXG5bKl0gLS0-IHMxXG5zMSAtLT4gczJcbnMyIC0tPiBzM1xuczMgLS0-IHM0XG5zNCAtLT4gczVcbnM1IC0tPiBzNlxuc3RhdGUgXCJjcmVhdGUgc2FtcGxlciBhbmQgc2FtcGxlciB2aWV3XCIgYXMgczFcbnN0YXRlIFwiY3JlYXRlIGRlc2NyaXB0b3IgaGFzaCBmb3Igc2FtcGxlciBhbmQgc2FtcGxlciB2aWV3XCIgYXMgczJcbnN0YXRlIFwicG9zc2libHkgaW52YWxpZGF0ZSBzYW1wbGVyIGRlc2NyaXB0b3Igc3RhdGVzXCIgYXMgczNcbnN0YXRlIFwiemlua19kcmF3X3Zib1wiIGFzIHM0XG5zdGF0ZSBcInVwZGF0ZV9kZXNjcmlwdG9yc1wiIGFzIHM1XG5zdGF0ZSBcInJlaGFzaCBzYW1wbGVyIHN0YXRlcyB1c2luZyBwcmUtaGFzaGVkIHZhbHVlc1wiIGFzIHM2XG4iLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCIsInRoZW1lVmFyaWFibGVzIjp7ImJhY2tncm91bmQiOiJ3aGl0ZSIsInByaW1hcnlDb2xvciI6IiNFQ0VDRkYiLCJzZWNvbmRhcnlDb2xvciI6IiNmZmZmZGUiLCJ0ZXJ0aWFyeUNvbG9yIjoiaHNsKDgwLCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJwcmltYXJ5Qm9yZGVyQ29sb3IiOiJoc2woMjQwLCA2MCUsIDg2LjI3NDUwOTgwMzklKSIsInNlY29uZGFyeUJvcmRlckNvbG9yIjoiaHNsKDYwLCA2MCUsIDgzLjUyOTQxMTc2NDclKSIsInRlcnRpYXJ5Qm9yZGVyQ29sb3IiOiJoc2woODAsIDYwJSwgODYuMjc0NTA5ODAzOSUpIiwicHJpbWFyeVRleHRDb2xvciI6IiMxMzEzMDAiLCJzZWNvbmRhcnlUZXh0Q29sb3IiOiIjMDAwMDIxIiwidGVydGlhcnlUZXh0Q29sb3IiOiJyZ2IoOS41MDAwMDAwMDAxLCA5LjUwMDAwMDAwMDEsIDkuNTAwMDAwMDAwMSkiLCJsaW5lQ29sb3IiOiIjMzMzMzMzIiwidGV4dENvbG9yIjoiIzMzMyIsIm1haW5Ca2ciOiIjRUNFQ0ZGIiwic2Vjb25kQmtnIjoiI2ZmZmZkZSIsImJvcmRlcjEiOiIjOTM3MERCIiwiYm9yZGVyMiI6IiNhYWFhMzMiLCJhcnJvd2hlYWRDb2xvciI6IiMzMzMzMzMiLCJmb250RmFtaWx5IjoiXCJ0cmVidWNoZXQgbXNcIiwgdmVyZGFuYSwgYXJpYWwiLCJmb250U2l6ZSI6IjE2cHgiLCJsYWJlbEJhY2tncm91bmQiOiIjZThlOGU4Iiwibm9kZUJrZyI6IiNFQ0VDRkYiLCJub2RlQm9yZGVyIjoiIzkzNzBEQiIsImNsdXN0ZXJCa2ciOiIjZmZmZmRlIiwiY2x1c3RlckJvcmRlciI6IiNhYWFhMzMiLCJkZWZhdWx0TGlua0NvbG9yIjoiIzMzMzMzMyIsInRpdGxlQ29sb3IiOiIjMzMzIiwiZWRnZUxhYmVsQmFja2dyb3VuZCI6IiNlOGU4ZTgiLCJhY3RvckJvcmRlciI6ImhzbCgyNTkuNjI2MTY4MjI0MywgNTkuNzc2NTM2MzEyOCUsIDg3LjkwMTk2MDc4NDMlKSIsImFjdG9yQmtnIjoiI0VDRUNGRiIsImFjdG9yVGV4dENvbG9yIjoiYmxhY2siLCJhY3RvckxpbmVDb2xvciI6ImdyZXkiLCJzaWduYWxDb2xvciI6IiMzMzMiLCJzaWduYWxUZXh0Q29sb3IiOiIjMzMzIiwibGFiZWxCb3hCa2dDb2xvciI6IiNFQ0VDRkYiLCJsYWJlbEJveEJvcmRlckNvbG9yIjoiaHNsKDI1OS42MjYxNjgyMjQzLCA1OS43NzY1MzYzMTI4JSwgODcuOTAxOTYwNzg0MyUpIiwibGFiZWxUZXh0Q29sb3IiOiJibGFjayIsImxvb3BUZXh0Q29sb3IiOiJibGFjayIsIm5vdGVCb3JkZXJDb2xvciI6IiNhYWFhMzMiLCJub3RlQmtnQ29sb3IiOiIjZmZmNWFkIiwibm90ZVRleHRDb2xvciI6ImJsYWNrIiwiYWN0aXZhdGlvbkJvcmRlckNvbG9yIjoiIzY2NiIsImFjdGl2YXRpb25Ca2dDb2xvciI6IiNmNGY0ZjQiLCJzZXF1ZW5jZU51bWJlckNvbG9yIjoid2hpdGUiLCJzZWN0aW9uQmtnQ29sb3IiOiJyZ2JhKDEwMiwgMTAyLCAyNTUsIDAuNDkpIiwiYWx0U2VjdGlvbkJrZ0NvbG9yIjoid2hpdGUiLCJzZWN0aW9uQmtnQ29sb3IyIjoiI2ZmZjQwMCIsInRhc2tCb3JkZXJDb2xvciI6IiM1MzRmYmMiLCJ0YXNrQmtnQ29sb3IiOiIjOGE5MGRkIiwidGFza1RleHRMaWdodENvbG9yIjoid2hpdGUiLCJ0YXNrVGV4dENvbG9yIjoid2hpdGUiLCJ0YXNrVGV4dERhcmtDb2xvciI6ImJsYWNrIiwidGFza1RleHRPdXRzaWRlQ29sb3IiOiJibGFjayIsInRhc2tUZXh0Q2xpY2thYmxlQ29sb3IiOiIjMDAzMTYzIiwiYWN0aXZlVGFza0JvcmRlckNvbG9yIjoiIzUzNGZiYyIsImFjdGl2ZVRhc2tCa2dDb2xvciI6IiNiZmM3ZmYiLCJncmlkQ29sb3IiOiJsaWdodGdyZXkiLCJkb25lVGFza0JrZ0NvbG9yIjoibGlnaHRncmV5IiwiZG9uZVRhc2tCb3JkZXJDb2xvciI6ImdyZXkiLCJjcml0Qm9yZGVyQ29sb3IiOiIjZmY4ODg4IiwiY3JpdEJrZ0NvbG9yIjoicmVkIiwidG9kYXlMaW5lQ29sb3IiOiJyZWQiLCJsYWJlbENvbG9yIjoiYmxhY2siLCJlcnJvckJrZ0NvbG9yIjoiIzU1MjIyMiIsImVycm9yVGV4dENvbG9yIjoiIzU1MjIyMiIsImNsYXNzVGV4dCI6IiMxMzEzMDAiLCJmaWxsVHlwZTAiOiIjRUNFQ0ZGIiwiZmlsbFR5cGUxIjoiI2ZmZmZkZSIsImZpbGxUeXBlMiI6ImhzbCgzMDQsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlMyI6ImhzbCgxMjQsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSIsImZpbGxUeXBlNCI6ImhzbCgxNzYsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlNSI6ImhzbCgtNCwgMTAwJSwgOTMuNTI5NDExNzY0NyUpIiwiZmlsbFR5cGU2IjoiaHNsKDgsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlNyI6ImhzbCgxODgsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSJ9fSwidXBkYXRlRWRpdG9yIjpmYWxzZX0)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic3RhdGVEaWFncmFtXG5bKl0gLS0-IHMxXG5zMSAtLT4gczJcbnMyIC0tPiBzM1xuczMgLS0-IHM0XG5zNCAtLT4gczVcbnM1IC0tPiBzNlxuc3RhdGUgXCJjcmVhdGUgc2FtcGxlciBhbmQgc2FtcGxlciB2aWV3XCIgYXMgczFcbnN0YXRlIFwiY3JlYXRlIGRlc2NyaXB0b3IgaGFzaCBmb3Igc2FtcGxlciBhbmQgc2FtcGxlciB2aWV3XCIgYXMgczJcbnN0YXRlIFwicG9zc2libHkgaW52YWxpZGF0ZSBzYW1wbGVyIGRlc2NyaXB0b3Igc3RhdGVzXCIgYXMgczNcbnN0YXRlIFwiemlua19kcmF3X3Zib1wiIGFzIHM0XG5zdGF0ZSBcInVwZGF0ZV9kZXNjcmlwdG9yc1wiIGFzIHM1XG5zdGF0ZSBcInJlaGFzaCBzYW1wbGVyIHN0YXRlcyB1c2luZyBwcmUtaGFzaGVkIHZhbHVlc1wiIGFzIHM2XG4iLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCIsInRoZW1lVmFyaWFibGVzIjp7ImJhY2tncm91bmQiOiJ3aGl0ZSIsInByaW1hcnlDb2xvciI6IiNFQ0VDRkYiLCJzZWNvbmRhcnlDb2xvciI6IiNmZmZmZGUiLCJ0ZXJ0aWFyeUNvbG9yIjoiaHNsKDgwLCAxMDAlLCA5Ni4yNzQ1MDk4MDM5JSkiLCJwcmltYXJ5Qm9yZGVyQ29sb3IiOiJoc2woMjQwLCA2MCUsIDg2LjI3NDUwOTgwMzklKSIsInNlY29uZGFyeUJvcmRlckNvbG9yIjoiaHNsKDYwLCA2MCUsIDgzLjUyOTQxMTc2NDclKSIsInRlcnRpYXJ5Qm9yZGVyQ29sb3IiOiJoc2woODAsIDYwJSwgODYuMjc0NTA5ODAzOSUpIiwicHJpbWFyeVRleHRDb2xvciI6IiMxMzEzMDAiLCJzZWNvbmRhcnlUZXh0Q29sb3IiOiIjMDAwMDIxIiwidGVydGlhcnlUZXh0Q29sb3IiOiJyZ2IoOS41MDAwMDAwMDAxLCA5LjUwMDAwMDAwMDEsIDkuNTAwMDAwMDAwMSkiLCJsaW5lQ29sb3IiOiIjMzMzMzMzIiwidGV4dENvbG9yIjoiIzMzMyIsIm1haW5Ca2ciOiIjRUNFQ0ZGIiwic2Vjb25kQmtnIjoiI2ZmZmZkZSIsImJvcmRlcjEiOiIjOTM3MERCIiwiYm9yZGVyMiI6IiNhYWFhMzMiLCJhcnJvd2hlYWRDb2xvciI6IiMzMzMzMzMiLCJmb250RmFtaWx5IjoiXCJ0cmVidWNoZXQgbXNcIiwgdmVyZGFuYSwgYXJpYWwiLCJmb250U2l6ZSI6IjE2cHgiLCJsYWJlbEJhY2tncm91bmQiOiIjZThlOGU4Iiwibm9kZUJrZyI6IiNFQ0VDRkYiLCJub2RlQm9yZGVyIjoiIzkzNzBEQiIsImNsdXN0ZXJCa2ciOiIjZmZmZmRlIiwiY2x1c3RlckJvcmRlciI6IiNhYWFhMzMiLCJkZWZhdWx0TGlua0NvbG9yIjoiIzMzMzMzMyIsInRpdGxlQ29sb3IiOiIjMzMzIiwiZWRnZUxhYmVsQmFja2dyb3VuZCI6IiNlOGU4ZTgiLCJhY3RvckJvcmRlciI6ImhzbCgyNTkuNjI2MTY4MjI0MywgNTkuNzc2NTM2MzEyOCUsIDg3LjkwMTk2MDc4NDMlKSIsImFjdG9yQmtnIjoiI0VDRUNGRiIsImFjdG9yVGV4dENvbG9yIjoiYmxhY2siLCJhY3RvckxpbmVDb2xvciI6ImdyZXkiLCJzaWduYWxDb2xvciI6IiMzMzMiLCJzaWduYWxUZXh0Q29sb3IiOiIjMzMzIiwibGFiZWxCb3hCa2dDb2xvciI6IiNFQ0VDRkYiLCJsYWJlbEJveEJvcmRlckNvbG9yIjoiaHNsKDI1OS42MjYxNjgyMjQzLCA1OS43NzY1MzYzMTI4JSwgODcuOTAxOTYwNzg0MyUpIiwibGFiZWxUZXh0Q29sb3IiOiJibGFjayIsImxvb3BUZXh0Q29sb3IiOiJibGFjayIsIm5vdGVCb3JkZXJDb2xvciI6IiNhYWFhMzMiLCJub3RlQmtnQ29sb3IiOiIjZmZmNWFkIiwibm90ZVRleHRDb2xvciI6ImJsYWNrIiwiYWN0aXZhdGlvbkJvcmRlckNvbG9yIjoiIzY2NiIsImFjdGl2YXRpb25Ca2dDb2xvciI6IiNmNGY0ZjQiLCJzZXF1ZW5jZU51bWJlckNvbG9yIjoid2hpdGUiLCJzZWN0aW9uQmtnQ29sb3IiOiJyZ2JhKDEwMiwgMTAyLCAyNTUsIDAuNDkpIiwiYWx0U2VjdGlvbkJrZ0NvbG9yIjoid2hpdGUiLCJzZWN0aW9uQmtnQ29sb3IyIjoiI2ZmZjQwMCIsInRhc2tCb3JkZXJDb2xvciI6IiM1MzRmYmMiLCJ0YXNrQmtnQ29sb3IiOiIjOGE5MGRkIiwidGFza1RleHRMaWdodENvbG9yIjoid2hpdGUiLCJ0YXNrVGV4dENvbG9yIjoid2hpdGUiLCJ0YXNrVGV4dERhcmtDb2xvciI6ImJsYWNrIiwidGFza1RleHRPdXRzaWRlQ29sb3IiOiJibGFjayIsInRhc2tUZXh0Q2xpY2thYmxlQ29sb3IiOiIjMDAzMTYzIiwiYWN0aXZlVGFza0JvcmRlckNvbG9yIjoiIzUzNGZiYyIsImFjdGl2ZVRhc2tCa2dDb2xvciI6IiNiZmM3ZmYiLCJncmlkQ29sb3IiOiJsaWdodGdyZXkiLCJkb25lVGFza0JrZ0NvbG9yIjoibGlnaHRncmV5IiwiZG9uZVRhc2tCb3JkZXJDb2xvciI6ImdyZXkiLCJjcml0Qm9yZGVyQ29sb3IiOiIjZmY4ODg4IiwiY3JpdEJrZ0NvbG9yIjoicmVkIiwidG9kYXlMaW5lQ29sb3IiOiJyZWQiLCJsYWJlbENvbG9yIjoiYmxhY2siLCJlcnJvckJrZ0NvbG9yIjoiIzU1MjIyMiIsImVycm9yVGV4dENvbG9yIjoiIzU1MjIyMiIsImNsYXNzVGV4dCI6IiMxMzEzMDAiLCJmaWxsVHlwZTAiOiIjRUNFQ0ZGIiwiZmlsbFR5cGUxIjoiI2ZmZmZkZSIsImZpbGxUeXBlMiI6ImhzbCgzMDQsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlMyI6ImhzbCgxMjQsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSIsImZpbGxUeXBlNCI6ImhzbCgxNzYsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlNSI6ImhzbCgtNCwgMTAwJSwgOTMuNTI5NDExNzY0NyUpIiwiZmlsbFR5cGU2IjoiaHNsKDgsIDEwMCUsIDk2LjI3NDUwOTgwMzklKSIsImZpbGxUeXBlNyI6ImhzbCgxODgsIDEwMCUsIDkzLjUyOTQxMTc2NDclKSJ9fSwidXBkYXRlRWRpdG9yIjpmYWxzZX0)

In this way, I could have a single 32bit value representing the sampler view that persisted for its lifetime, and a pair of 32bit values for the sampler (since I still need to potentially toggle between linear and nearest filtering) that I can select between. This ends up being 8bytes to hash, which is over 50% less. It's not a huge change in the flamegraph, but it's possibly an interesting factoid. Also, as the layouts will always be the same for these descriptors, that member can safely be omitted from the original sampler_view hash.

## Set Usage
Another vaguely interesting tidbit is profiling zink's usage of mesa's `set` implementation, which is used to ensure that various objects are only added to a given batch a single time. Historically the pattern for use in zink has been something like:
```c
if (!_mesa_set_search(set, value)) {
   do_something();
   _mesa_set_add(set, value);
}
```
This is not ideal, as it ends up performing two lookups in the `set` for cases where the value isn't already present. A much better practice is:
```c
bool found = false;
_mesa_set_search_and_add(set, value, &found);
if (!found)
    do_something();
```
In this way, the lookup is only done once, which ends up being huge for large sets.

## For My Final Trick
I was now holding steady at **33fps**, but there was a tiny bit more performance to squeeze out of descriptor updating when I began to analyze how much looping was being done. This is a general overview of all the loops in `update_descriptors()` for each type of descriptor at the time of my review:
* loop for all shader stages
  * loop for all bindings in the shader
    * in sampler and image bindings, loop for all resourecs in the binding
* loop for all resources in descriptor set
* loop for all barriers to be applied in descriptor set

This was a lot of looping, and it was especially egregious in the final component of my refactored `update_descriptors():
```c
static bool
write_descriptors(struct zink_context *ctx, struct zink_descriptor_set *zds, unsigned num_wds, VkWriteDescriptorSet *wds,
                 unsigned num_resources, struct zink_descriptor_resource *resources, struct set *persistent,
                 bool is_compute, bool cache_hit)
{
   bool need_flush = false;
   struct zink_batch *batch = is_compute ? &ctx->compute_batch : zink_curr_batch(ctx);
   struct zink_screen *screen = zink_screen(ctx->base.screen);
   assert(zds->desc_set);
   unsigned check_flush_id = is_compute ? 0 : ZINK_COMPUTE_BATCH_ID;
   for (int i = 0; i < num_resources; ++i) {
      assert(num_resources <= zds->pool->num_resources);

      struct zink_resource *res = resources[i].res;
      if (res) {
         need_flush |= zink_batch_reference_resource_rw(batch, res, resources[i].write) == check_flush_id;
         if (res->persistent_maps)
            _mesa_set_add(persistent, res);
      }
      /* if we got a cache hit, we have to verify that the cached set is still valid;
       * we store the vk resource to the set here to avoid a more complex and costly mechanism of maintaining a
       * hash table on every resource with the associated descriptor sets that then needs to be iterated through
       * whenever a resource is destroyed
       */
      assert(!cache_hit || zds->resources[i] == res);
      if (!cache_hit)
         zink_resource_desc_set_add(res, zds, i);
   }
   if (!cache_hit && num_wds)
      vkUpdateDescriptorSets(screen->dev, num_wds, wds, 0, NULL);
   for (int i = 0; zds->pool->num_descriptors && i < util_dynarray_num_elements(&zds->barriers, struct zink_descriptor_barrier); ++i) {
      struct zink_descriptor_barrier *barrier = util_dynarray_element(&zds->barriers, struct zink_descriptor_barrier, i);
      zink_resource_barrier(ctx, NULL, barrier->res,
                            barrier->layout, barrier->access, barrier->stage);
   }

   return need_flush;
}
```
This function iterates over all the resourecs in a descriptor set, tagging them for batch usage and persistent mapping, adding references for the descriptor set to the resource as I previously delved into. Then it iterates over the barriers and applies them.

But why was I iterating over all the resources and then over all the barriers when every resource will always have a barrier for the descriptor set, even if it ends up getting filtered out based on previous usage?

It just doesn't make sense.

So I refactored this a bit, and now there's only one loop:
```c
static bool
write_descriptors(struct zink_context *ctx, struct zink_descriptor_set *zds, unsigned num_wds, VkWriteDescriptorSet *wds,
                 struct set *persistent, bool is_compute, bool cache_hit)
{
   bool need_flush = false;
   struct zink_batch *batch = is_compute ? &ctx->compute_batch : zink_curr_batch(ctx);
   struct zink_screen *screen = zink_screen(ctx->base.screen);
   assert(zds->desc_set);
   unsigned check_flush_id = is_compute ? 0 : ZINK_COMPUTE_BATCH_ID;

   if (!cache_hit && num_wds)
      vkUpdateDescriptorSets(screen->dev, num_wds, wds, 0, NULL);
   for (int i = 0; zds->pool->num_descriptors && i < util_dynarray_num_elements(&zds->barriers, struct zink_descriptor_barrier); ++i) {
      struct zink_descriptor_barrier *barrier = util_dynarray_element(&zds->barriers, struct zink_descriptor_barrier, i);
      if (barrier->res->persistent_maps)
         _mesa_set_add(persistent, barrier->res);
      need_flush |= zink_batch_reference_resource_rw(batch, barrier->res, zink_resource_access_is_write(barrier->access)) == check_flush_id;
      zink_resource_barrier(ctx, NULL, barrier->res,
                            barrier->layout, barrier->access, barrier->stage);
   }

   return need_flush;
}
```
This actually has the side benefit of reducing the required looping even further, as barriers get merged based on access and stages, meaning that though there may be `N` resources used by a given set used by `M` stages, it's possible that the looping here might be reduced to only `N` rather than `N * M` since all barriers might be consolidated.

## In Closing
Let's check all the changes out in the flamegraph:
[![final.png]({{site.url}}/assets/desc_profiling1/final.png)]({{site.url}}/assets/desc_profiling1/final.png)

This last bit has shaved off another big chunk of CPU usage overall, bringing `update_descriptors()` from 11.4% to 9.32%. Descriptor state updating is down from 0.718% to 0.601% from the pre-hashing as well, though this wasn't exactly a huge target to hit.

Just for nostalgia, here's the starting point from just after I'd split the descriptor types into different sets and we all thought 27fps with descriptor set caching was a lot:
[![split.png]({{site.url}}/assets/desc_profiling1/split.png)]({{site.url}}/assets/desc_profiling1/split.png)

But now zink is up another 25% performance to a steady **34fps**:
[![heaven.png]({{site.url}}/assets/desc_profiling1/heaven.png)]({{site.url}}/assets/desc_profiling1/heaven.png)

And I did it in only four blog posts.

## But Then, The Future
Looking forward, there's still some easy work that can be done here.

For starters, I'd probably improve descriptor states a little such that I also had a flag anytime the batch cycled. This would enable me to add batch-tracking for resources/samplers/sampler_views more reliably when was actually necessary vs trying to add it every time, which ends up being a significant perf hit from all the lookups. I imagine that'd make a huge part of the remaining `update_descriptors()` usage disappear.
[![future-lookups.png]({{site.url}}/assets/desc_profiling1/future-lookups.png)]({{site.url}}/assets/desc_profiling1/future-lookups.png)

There's also, as ever, the pipeline hashing, which can further be reduced by adding more dynamic state handling, which would remove more values from the pipeline state and thus reduce the amount of hashing required.
[![future-pipeline.png]({{site.url}}/assets/desc_profiling1/future-pipeline.png)]({{site.url}}/assets/desc_profiling1/future-pipeline.png)

I'd probably investigate doing some resource caching to keep a bucket of destroyed resources around for faster reuse since there's a fair amount of that going on.

