---
published: false
---
## SP33D

I was planning to write this Friday, but then it was Friday so I didn't.

You know how it is.

I've been doing a lot of work on CPU optimizations in zink lately. I had planned to do some benchmarks of this, but now it's Monday and [someone has already done it for me](https://www.phoronix.com/review/zink-radeon-august2022), so I won't.

Sometimes it's like that too.

But the overly-technical, word-heavy, bullet-point-laden blog post still needs to be written, and now it's Monday, so here I am.

Speed: How does it work?

## Descriptors: Recap

I've blogged a lot about descriptors in the past. After years of pointlessly churning the codebase, a winner has emerged from the manager wars. Descriptor caching has been [deleted](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18051). It's gone, and laziness is the future. Huzzah.

To recap for those who haven't followed however many posts I've written on the topic, the idea of "lazy" descriptors is to do the least amount of work as stupidly as possible:
* split descriptors into 6 sets:
  * uniforms
  * ubos
  * textures
  * ssbosIt's free real estate.
  * images
  * bindless
* bucket allocate tons of descriptor sets alongside each cmdbuf
* do a [templated](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_descriptor_update_template.html) full set update any time a descriptor for a given type changes
* bind new set
* do no other work

Simple, yet surprisingly performant.

## Descriptors: Faster
We know that laziness is preferable to doing any kind of work. This is an indisputable fact. It's also (obviously) true in CPU performance: less work is better. So how can zink do even less work than being lazy?

Well.

It's always possible to be lazier.

Historically, there have been two points of non-laziness when it comes to lazy descriptors:
* descriptor pool lookup
* descriptor set allocation

Descriptor sets are allocated from descriptor pools. Descriptor pools are allocated based on the pipeline layout. Pools are allocated onto the same struct that contains the cmdbuf, ensuring that the lifetimes match. Thus, for each pipeline layout, there are `N` descriptor pools, where `N` is the number of cmdbufs in use. Any time a new set must be allocated, the corresponding pool must be accessed. To access the the pool, the integer ID of the pipeline layout is used, as all the pools for a given cmdbuf are stored in a hash table.

Hash tables are slow. They require hashing, they require lookups, and why not just use an array if the key is already an integer? It's a mystery, especially when there aren't thousands of items, so I switched to using an array in order to enable direct lookups.

**Performance: enhanced.**

Descriptor set allocation was another bottleneck. In order to avoid blowing out heaps on [heap-based hardware](https://www.jlekstrand.net/jason/blog/2022/08/descriptors-are-hard/), I've capped descriptor pools to only contain 100 sets at a time. This means that even if a set isn't fully utilized, it's not consuming a huge amount of resources. It also means that allocation is faster when a cmdbuf (and thus its associated descriptor pools) gets reset.

Remember when I said that there were `N` descriptor pools for `N` cmdbufs? Obviously this was a lie. What I meant to say was there are `N * O` descriptor pools for `N` cmdbufs, where `O` is the number of times the descriptor pool has overflowed because it had to allocate more than 100 sets. In this scenario, the overflowed (full) descriptor pool is appended to an array which then gets freed upon cmdbuf reset. Since the pools are relatively small, this recycling operation is pretty fast.

But it's still an operation. Which means it's doing work. ~~And you know what I hate?~~ I mean, *You know what CPU performance hates*? Doing work. Yeah, that's it.

So now instead of recycling these overflowed pools, zink just retains them and reuses them directly.

[![memory-meme.png]({{site.url}}/assets/memory-meme.png)]({{site.url}}/assets/memory-meme.png)

**Performance: enhanced.**

## More SP33D
Join me in the next blog post when I show how it's possible for anyone (even you!) to turn code into spaghetti using C++.