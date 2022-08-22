---
published: false
---
## Sp33d

I was planning to write this Friday, but then it was Friday so I didn't.

You know how it is.

I've been doing a lot of work on CPU optimizations in zink lately. I had planned to do some benchmarks of this, but now it's Monday and [someone has already done it for me](https://www.phoronix.com/review/zink-radeon-august2022), so I won't.

Sometimes it's like that too.

But the overly-technical, word-heavy blog post still needs to be written, and now it's Monday, so here I am.

Speed: How does it work?

## Descriptors

I've blogged a lot about descriptors in the past. After years of pointlessly churning the codebase, a winner has emerged from the manager wars. Descriptor caching has been [deleted](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18051). It's gone, and laziness is the future. Huzzah.

To recap for those who haven't followed however many posts I've written on the topic, the idea of "lazy" descriptors is to do the least amount of work as stupidly as possible:
* split descriptors into 6 sets:
  * uniforms
  * ubos
  * textures
  * ssbos
  * images
  * bindless
* bucket allocate tons of descriptor sets
* do a [templated](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_descriptor_update_template.html) full set update any time a descriptor for a given type changes
* bind new set
* do no other work

Simple, yet surprisingly performant.