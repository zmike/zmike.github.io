---
published: false
---
## I Said I Would

A long, long time ago in a month far, far away I said I was going to blog about some improvements I'd been working on for zink. I blogged about some of them, but one was conspicuously absent from the original list:
* make zink usable for gaming

There's a lot that goes into this item. The post you're reading now isn't about to go so far as to claim that zink(-wip) is usable for gaming. No, that day is still far, far away. But this post is going to be the first step.

To begin with, a riddle: what change was made to zink between these two screenshots?

[![tr-slow.png]({{site.url}}/assets/tr-slow.png)]({{site.url}}/assets/tr-slow.png)

[![tr-zoom.png]({{site.url}}/assets/tr-zoom.png)]({{site.url}}/assets/tr-zoom.png)

That's right, I put the punchline in the title.

A suballocator.

## What Is A Suballocator?
A suballocator is a mechanism by which small blocks of memory can be suballocated out of larger one. For example, if I want to allocate an 64byte chunk of memory, I could allocate it directly and get my block, or I could allocate a 4096byte chunk of memory and then take 64bytes out of it. 

When performance is involved, it's important to consider the time-cost of allocations, and so yes, it's useful to have already allocated another 63 instances of 64bytes when I need a second one, but there's another, deeper issue that's also necessary to address, especially as it relates to gaming: 32bit environments.

In a 32bit process, the amount of address space available is limited to 4GB, regardless of how much actual memory is physically present, some of which is dedicated to system resources and unavailable for general use. Any time a buffer or image is mapped by the driver in a process, this uses up address space in order to create an addressable region of memory that can be read or written to. Once all the address space has been used up, no other resources can be mapped, and it becomes impossible to continue normal operations.

In short, the game crashes.

In Vulkan, and just generally in driver work, it's important to keep allocation sizes aligned to the preference of the hardware for a given usage; this amounts to [minMemoryMapAlignment](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDeviceLimits.html), which is 4096bytes on many drivers. Similarly, `vkGetBufferMemoryRequirements` and `vkGetImageMemoryRequirements` return aligned memory sizes, so even if only 64bytes are needed, 4096bytes must still be allocated. This ends up wasting tons of memory when an app is allocating lots of smaller regions, and it's further wasting address space since Vulkan prohibits memory from being mapped multiple times, meaning that each 64byte buffer is wasting an additional 4032bytes of address space.