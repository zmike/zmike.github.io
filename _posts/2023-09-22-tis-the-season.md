---
published: false
---
# Remember Way Back When...

This blog was about pointlessly optimizing things? I'm talking like taking [vkGetDescriptorSetLayoutSupport](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkGetDescriptorSetLayoutSupport.html) and making it fast. The kinds of optimizations nobody asked for and potentially nobody even wanted.

Well good news: this isn't a post about those types of optimizations.

This is a post where I'm gonna talk about some speedups that you didn't even know you craved but now that you know they exist you can't live without.

# The Vulkan Queue: What Is It?
Lots of people are asking, but surely nobody reading this blog since you're all experts. But if you have a friend who wants to know, here's [the official resource](https://registry.khronos.org/vulkan/site/guide/latest/queues.html) for all that knowledge. It's got diagrams. Images with important parts circled. The stuff that means whoever wrote it knew what they were talking about.

The thing this "official" resource doesn't tell you is the queue is potentially pretty slow. You chuck some commands into it, and then you wait on your fence/semaphore, but the actual time it takes to perform queue submission is nonzero. In fact, it's quite a bit larger than zero. *How large is it?* you might be asking.

I didn't want to do this, but you've forced my hand.

# The Vulkan Queue: Timing

What if I told you there was a tool for measuring things like this. A tool for determining the cost of various Vulkan operations. For *benchmarking*, one might say.

That's right, it's time to yet again plug [vkoverhead](https://github.com/zmike/vkoverhead), the best and only tool for doing whatever I'm about to do.

Like a prophet, my past self already predicted that I'd be sitting here writing this post to close out a week of `types_of_headaches.meme -> vulkan_headaches.meme`. That's why `vkoverhead` already has the `-submit-only` option in order to run a series of benchmark cases which have numbers that are totally not about to go up.

Let's look at those cases now to fill up some more page space and time travel closer to the end of my workweek:
* `submit_noop` submits nothing. There's no semaphores, no cmdbufs, it just submits and returns in order to provide a baseline
* `submit_50noop` submits nothing 50 times, which is to say it passes 50x `VkSubmitInfo` structs to `vkQueueSubmit` (or the 2 versions if sync2 is supported)
* `submit_1cmdbuf` submits a single cmdbuf. In theory this should be slower than the noop case, but I hate computers and obviously this isn't true at all
* `submit_50cmdbuf` submits 50 cmdbufs. In theory this should be slower than the single cmdbuf case, and, thankfully, this one particular time in which we have expectations of how computers work does match our expectations, and the numbers are lower
* `submit_50cmdbuf_50submit` submits 50 cmdbufs in 50 submits for a total of 2500 cmdbufs per `vkQueueSubmit` call. This is the slowest test, you would think, and the longer 