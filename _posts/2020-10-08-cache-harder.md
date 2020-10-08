---
published: false
---
## Descriptors Once More

It's just that kind of week.

When I left off in my last post, I'd just implemented a two-tiered cache system for managing descriptor sets which was objectively worse in performance than not doing any caching at all.

Cool.

Next, I did some analysis of actual descriptor usage, and it turned out that the UBO churn was massive, while sampler descriptors were only changed occasionally. This is due to a mechanism in mesa involving a NIR pass which rewrites uniform data passed to the OpenGL context as UBO loads in the shader, compacting the data into a single buffer and allowing it to be more efficiently passed to the GPU. There's a utility component `u_upload_mgr` for gallium-based drivers which allocates a large (~100k) buffer and then maps/writes to it at offsets to avoid needing to create a new buffer for this every time the uniform data changes.

The downside of `u_upload_mgr`, for the current state of zink, is that it means the hashed descriptor states are different for almost every single draw because while the UBO doesn't actually change, the offset does, and this is necessarily part of the descriptor hash since zink is exclusively using `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER` descriptors.

I needed to increase the efficiency of the cache to make it worthwhile, so I decided first to reduce the impact of a changed descriptor state hash during pre-draw updating. This way, even if the UBO descriptor state hash was changing every time, maybe it wouldn't be so impactful.

## Deep Cuts
What this amounted to was to split the giant, all-encompassing descriptor set, which included UBOs, samplers, SSBOs, and shader images, into separate sets such that each type of descriptor would be isolated from changes in the other descriptors between draws.

Thus, I now had four distinct descriptor pools for each program, and I was producing and binding four descriptor sets for every draw. I also changed the shader compiler a bit to always bind shader resources to the newly-split sets as well as created some dummy descriptor sets since it's illegal to bind sets to a command buffer with non-sequential indices, but it was mostly easy work. It seemed like a great plan, or at least one that had a lot of potential for more optimizations based on it. As far as direct performance increases from the split, UBO descriptors would be constantly changing, but maybe...

Well, the patch is pretty big (8 files changed, 766 insertions(+), 450 deletions(-)), but in the end, I was still stuck around 23 fps.

## Dynamic UBOs
With this work done, I decided to switch things up a bit and explore using `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC` descriptors for the mesa-produced UBOs, as this would reduce both the hashing required to calculate the UBO descriptor state (offset no longer needs to be factored in, which is one fewer `uint32_t` to hash) as well as cache misses due to changing offsets.

Due to potential driver limitations, only these mesa-produced UBOs are dynamic now, otherwise zink might exceed the [maxDescriptorSetUniformBuffersDynamic](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDeviceLimits.html) limit, but this was still more than enough.

Boom.

I was now at 27fps, the same as raw bucket allocation.

## Hash Performance
I'm not going to talk about hash performance. Mesa uses [xxhash](https://github.com/Cyan4973/xxHash) internally, and I'm just using the tools that are available.

What I am going to talk about, however, is the amount of hashing and lookups that I was doing.
 
Let's take a look at some [flamegraphs](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).
 
 [![meme-update_descriptors.png]({{site.url}}/assets/meme-update_descriptors.png)]({{site.url}}/assets/meme-update_descriptors.png)