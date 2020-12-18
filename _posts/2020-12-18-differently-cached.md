---
published: false
---
## memcpy harder

Amidst the flurry of patches being smashed into the repo today I thought I'd talk about about `memcpy`. Yes, I'm referring to the same function everyone knows and loves.

The final frontier of RPCS3 performance was `memcpy`. With ARGB emulation in place, my perf results were looking like this on RADV:

[![zink.png]({{site.url}}/assets/bioshock/zink.png)]({{site.url}}/assets/bioshock/zink.png)

Pushing a hard 12fps, this was a big improvement from before, but it seemed a bit low. Not having any prior experience in PS3 emulation, I started wondering whether this was the limit of things in the current year.

Meanwhile, RadeonSI was getting significantly higher performance with a graph like this:

[![radeonsi.png]({{site.url}}/assets/bioshock/radeonsi.png)]({{site.url}}/assets/bioshock/radeonsi.png)

Clearly performance is capable of being much higher, so why is zink so much worse at `memcpy`?

## Further Exploration...
Along with the wisdom of the great sage Dave Airlie led me to checking out the resource creation code in zink, specifically the types of memory that are used for allocation. Vulkan supports [a range of memory types](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkMemoryPropertyFlagBits.html) for the discerning allocations connoisseur, but the driver was jamming everything into `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`. This is great for ease of use, as the contents of buffers are always synchronized between CPU and GPU with no additional work needed, but it ends up being massively slower for any kind of direct copy operations of the backing memory, e.g., anything PBO-related.

What should really be used is `VK_MEMORY_PROPERTY_HOST_CACHED_BIT` whenever possible. This requires some additional legwork to properly invalidate/flush the memory used by any `vkMapMemory` calls, but the results were well worth the effort:

[![zink2.png]({{site.url}}/assets/bioshock/zink2.png)]({{site.url}}/assets/bioshock/zink2.png)

And performance was a buttery smooth 30fps (the cap) as well:

[![bioshock.png]({{site.url}}/assets/bioshock/bioshock.png)]({{site.url}}/assets/bioshock/bioshock.png)

Mission accomplished.