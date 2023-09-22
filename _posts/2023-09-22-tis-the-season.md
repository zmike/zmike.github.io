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
* `submit_50cmdbuf_50submit` submits 50 cmdbufs in 50 submits for a total of 50 cmdbufs per `vkQueueSubmit` call. This is the slowest test, you would think, and I thought that too, and the longer this explanation goes on the more you start to wonder if computers really do work at all like you expect or if this is going to upset you, but it's Friday, and I don't have anywhere to be except the gym, so I could keep delaying the inevitable for a while longer, but I do have to get to the gym so sure, this is totally gonna be way slower than all the other tests

It's a great series of tests which showcase some driver pain points. Specifically it shows how slow submit can be.

Let's check out some baseline results on the driver everyone loves to hang out with, RADV:
```
  40, submit_noop,                                        19569683,     100.0%
  41, submit_50noop,                                      402324,       2.1%
  42, submit_1cmdbuf,                                     51356,        0.3%
  43, submit_50cmdbuf,                                     1840,         0.0%
  44, submit_50cmdbuf_50submit,                            1031,         0.0%
```

Everything looks like we'd expect. The benchmark results ensmallen as they get more complex.

But why?

# Why So Slow
Because if you think about it like a smart human and not a dumb pile of "thinking" sand, submitting 50 cmdbufs is submitting 50 cmdbufs no matter how you do it.

[![queue-think.png]({{site.url}}/assets/queue-think.png)]({{site.url}}/assets/queue-think.png)

Some restrictions apply, signal semaphores blah blah blah, but none of that's happening here so what the fuck, RADV?

This is where we get into some real facepalm territory. Vulkan, as an API, gives drivers the ability to optimize this. That's the entire reason why [vkQueueSubmit](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkQueueSubmit.html) has the `submitCount` param and takes an array of submits.

But what does Mesa do here? Well, in [the current code](https://gitlab.freedesktop.org/mesa/mesa/-/blob/3c4c263dc734ec75f72d36b1d0d1a9cd41310112/src/vulkan/runtime/vk_queue.c#L1164) there's this gem:

```c
for (uint32_t i = 0; i < submitCount; i++) {
   struct vulkan_submit_info info = {
      .pNext = pSubmits[i].pNext,
      .command_buffer_count = pSubmits[i].commandBufferInfoCount,
      .command_buffers = pSubmits[i].pCommandBufferInfos,
      .wait_count = pSubmits[i].waitSemaphoreInfoCount,
      .waits = pSubmits[i].pWaitSemaphoreInfos,
      .signal_count = pSubmits[i].signalSemaphoreInfoCount,
      .signals = pSubmits[i].pSignalSemaphoreInfos,
      .fence = i == submitCount - 1 ? fence : NULL
   };
   VkResult result = vk_queue_submit(queue, &info);
   if (unlikely(result != VK_SUCCESS))
      return result;
}
```

Tremendous. It's worth mentioning that not only is this splitting the batched submits into individual ones, each submit also allocates a struct to contain the submit info so that the drivers can use the same interface. So not only is this increasing the kernel overhead by submitting more work there, it's also increasing memory bandwidth.

# Fast Forward
We've all been here before on SGC, and I really do need to get to the gym, so I'm authorizing a one-time fast forward to the results of optimizing this:

RADV GFX11:
```
  40, submit_noop,                                        19569683,     100.0%
  41, submit_50noop,                                      402324,       2.1%
  42, submit_1cmdbuf,                                     51356,        0.3%
  43, submit_50cmdbuf,                                     1840,         0.0%
  44, submit_50cmdbuf_50submit,                            1031,         0.0%
↓
  40, submit_noop,                                        21008648,     100.0%
  41, submit_50noop,                                      4866415,      23.2%
  42, submit_1cmdbuf,                                     51294,        0.2%
  43, submit_50cmdbuf,                                     1823,         0.0%
  44, submit_50cmdbuf_50submit,                            1828,         0.0%
```

That's like 1000% faster for case #41 and 50% faster for case #44.

*But how does this affect other drivers?* I'm sure you're asking next. And of course, this being the primary blog for distributing Mesa benchmarking numbers in any given year, I have those numbers.

Lavapipe:

```
  40, submit_noop,                                        1972672,      100.0%
  41, submit_50noop,                                      40334,        2.0%
  42, submit_1cmdbuf,                                     5994597,      303.9%
  43, submit_50cmdbuf,                                    2623720,      133.0%
  44, submit_50cmdbuf_50submit,                           133453,       6.8%
↓
  40, submit_noop,                                        1980681,      100.0%
  41, submit_50noop,                                      1202374,      60.7%
  42, submit_1cmdbuf,                                     6340872,      320.1%
  43, submit_50cmdbuf,                                    2482127,      125.3%
  44, submit_50cmdbuf_50submit,                           1165495,      58.8%
```

3000% faster for #41 and 1000% faster for #44.


Intel DG2:

```
  40, submit_noop,                                        101336,       100.0%
  41, submit_50noop,                                       2123,         2.1%
  42, submit_1cmdbuf,                                     35372,        34.9%
  43, submit_50cmdbuf,                                      713,          0.7%
  44, submit_50cmdbuf_50submit,                             707,          0.7%
↓
  40, submit_noop,                                        106065,       100.0%
  41, submit_50noop,                                      105992,       99.9%
  42, submit_1cmdbuf,                                     35110,        33.1%
  43, submit_50cmdbuf,                                      709,          0.7%
  44, submit_50cmdbuf_50submit,                             702,          0.7%
```

5000% faster for #41 and a big 🤕 for #44.


Turnip A740:
```
  40, submit_noop,                                        1227546,      100.0%
  41, submit_50noop,                                      26194,        2.1%
  42, submit_1cmdbuf,                                     1186327,      96.6%
  43, submit_50cmdbuf,                                    545341,       44.4%
  44, submit_50cmdbuf_50submit,                           16531,        1.3%
↓
  40, submit_noop,                                        1313550,      100.0%
  41, submit_50noop,                                      1078383,      82.1%
  42, submit_1cmdbuf,                                     1129515,      86.0%
  43, submit_50cmdbuf,                                    329247,       25.1%
  44, submit_50cmdbuf_50submit,                           484241,       36.9%
```

4000% faster for #41, 3000% faster for #44.

Pretty good, and it somehow manages to still be conformant.

Code [here](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25352).