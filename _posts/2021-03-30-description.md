---
published: false
---
## Yeah, Again

It's been a while since I blogged about descriptors, so let's fix that since this site may as well be called Super Good Descriptors.

Last time, I talked a bit about descriptors 3.0: lazy descriptors. The idea I settled on here was to do templated updates, and to do the absolute minimal amount of work possible for each draw while still never reusing any written-to descriptor sets once they'd been cycled out.

Lazy descriptors worked out great, and I'm sure many of you have grown to enjoy the `ZINK: USING LAZY DESCRIPTORS` log message that got spammed on startup over the past couple months of zink-wip.

Well, times have changed, and this message will no longer fill your terminal by default after today's (20210330) zink-wip snapshot.

## Modes
New today is the `ZINK_DESCRIPTORS` environment variable, supporting three modes of operation:
* `auto` - the default, which attempts to detect system capabilities and use caching with templated updates
* `lazy` - the mode we all know and love from the past few months
* `notemplates` - this is the old-style caching mechanism

`auto` mode moves zink one step closer to my eventual goal, which is to be able to use driconf to do application-based mode changes to disable caching for apps which never reuse resources. Also potentially useful would be the ability to dynamically disable caching on a pipeline-by-pipeline basis while an application is running if too many cache misses are detected.

## Necessary?
With that said, I've come to the conclusion that any form of caching may actually be, at best, equivalent to uncached mode for the general desktop user, and it may only be worthwhile for special cases, like Vulkan drivers which can't do descriptor templates or embedded devices. In my latest testing (on desktop systems), I have yet to see any scenarios where `lazy` mode fails to provide the best performance.

ARM in particular seems to [gain a lot from it](https://community.arm.com/developer/tools-software/graphics/b/blog/posts/vulkan-descriptor-and-buffer-management), as the post shows a ~40% perf improvement. It's unclear to me, however, whether any benchmarking was done against a highly optimized uncached implementation like I've done. The overhead from doing basic descriptor updating without templates is definitely significant, so it might just be that things are different now that more functionality is available. On Linux systems, at least, every Vulkan driver that matters supports descriptor templates, so this is functionality that can be relied upon.

## Is zink-wip slow now?
No.

The goal of zink-wip is to provide an optimal testing environment with the absolute bleeding edge in terms of performance and features. The `auto` mode should provide that, and the cases I've seen where its performance is noticeably worse number exactly one, and it's a subtest for `drawoverhead`. If anyone finds any other cases where `auto` is worse than `lazy`, I'm interested, but it shouldn't be a concern.

With that said, it might be worth doing some benchmarking between the two for some extremely high CPU usage scenarios, as that's the only case where it may be possible to detect a difference. Gone are the days of zink(-wip) hogging the whole CPU, so probably this is just useless pontificating to fill more of a blog page.

But also, if you're doing any kind of benchmarking on a high-end CPU, I'd probably recommend going with the `lazy` mode for now.

## Game-changers
I'm pleased with the current state of descriptor caching, but it does bother me that it isn't dramatically better than the uncached mode on my desktop. I think this ultimately just comes down to the current cache implementation being split into two steps:
* compute the descriptor cache key
* lookup the set

This effectively sits on top of the `lazy` mode, moving work out of the Vulkan driver and into the cache lookup any time there's a cache hit. As such, I've been considering working to shift some of this work into threads, though this is somewhat challenging given the current gallium API. Specifically, only SSBOs and shader images can be per-stage updated immediately after bind, as both UBOs and samplers bind only a single descriptor slot at a time, meaning there's no way to know when the "final" one is bound.

But then again, I've certainly reached the point of diminishing returns now. Most applications that I test have minimal CPU usage for the zink driver thread (e.g., Unigine Superposition is only at about 10% utilization on a i7-6700K), and are instead bottlenecking hard in the GPU, so I think it's time to call things "good enough" here unless things change.