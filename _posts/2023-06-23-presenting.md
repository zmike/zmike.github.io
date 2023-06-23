---
published: true
---
# When perf Is Too Slow

Keen-eyed readers of the blog will have noticed over the past couple weeks an obvious omission in my furious benchmark-related handwaving.

I didn't mention glmark2.

[![vulkan.svgz](https://phoronix.com/benchmark/result/radeonsi-vs-zink-opengl-benchmarks-2023/glmark2-1920-x-1080.svgz)](https://phoronix.com/benchmark/result/radeonsi-vs-zink-opengl-benchmarks-2023/glmark2-1920-x-1080.svgz)

Yes, that glmark2.

Except my results were a bit different.

The thing about glmark2 is it's just a CPU benchmark of drivers at this point. None of the rendering is in any way intensive, GPUs have gotten far more powerful than they were at the time the thing was written, and CPUs are now pushing tens of thousands of frames per second.

Yes, tens of thousands.

I set out to investigate this using the case with the biggest perf delta, which was `build:use-vbo=true`:
* **RadeonSI** - 25,318fps
* **Zink** - 10,419fps

Brutal.

Naturally I went to my standby for profiling, `perf`. I found some stuff, and I fixed it all, and surprise surprise the numbers didn't change. Clearly there was some blocking going on that was affecting the frames. Could the flamegraph tools for `perf` highlight this issue for me?

No.

An average Monday in the life of a zink developer.

New tools would be needed to figure out what's going on here.

Do such tools already exist? Maybe. Post in the comments if you have tools you use for this type of thing.

# WSI Can't This Code Be Faster
The solution I settled on is the olde-timey method of instrumentation. That is, injecting `os_time_get_nano()` calls in areas of interest and printing the diffs.

Let's take a moment and speculate on what my findings might have been.

Yes, obviously it's WSI.

It turns out **Mesa's (X11) WSI is too slow to push more than 10,000 fps**.

## Problems Acquired
As I've been tracking in [this issue](https://gitlab.freedesktop.org/mesa/mesa/-/issues/9201), the Mesa WSI code is very inefficient. This probably won't affect most things, but when the target frametime is around 40,000 *nanoseconds*, things get dicey.

For those of you unwilling to sort through your memorized table of frame timings, an application rendering at 60 fps must render frames in 16,666,667 nanoseconds.

glmark2 is over 400 times faster.

To enumerate the slow points I found at a high level:
* `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` is doing unlimited X11 roundtrips
* present is doing X11 roundtrips
* WSI common code is redoing (relatively) expensive ops on every acquire/present

Again, these aren't things that affect most apps/games, but when the frametime diff is 100,000ns vs 40,000ns, these add up to a majority of that difference.

## Brass Tacks: Surface Geometry
Going in order, part of the problem is the split nature of zink's WSI handling. The Gallium DRI frontend runs in a different thread from the zink driver thread, and it has its own integration with the API state tracker. This is fine for other drivers, as it lets relatively expensive WSI operations (e.g., display server roundtrips) occur without impeding the driver thread.

But how does this work with zink?

Zink uses the kopper interface, which is just a bridge between the Gallium DRI frontend and Vulkan WSI. In short, when DRI would directly call through to acquire/present, it instead handwaves some API methods at zink to maybe do the same thing.

One of these operations is updating the drawable geometry to determine the sizing of the framebuffer. In VK parlance, this is the image extents returned by `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`.

Now, looking at the other side, usually the DRI frontend (e.g., with RadeonSI) uses X11 configure notify events to determine the geometry, which lets it work asynchronously without roundtripping. Unfortunately, this doesn't work so well for zink, as there's no guarantee the underlying Vulkan driver will have the same idea of geometry. Thus, any time geometry is updated, the DRI frontend has to call through to zink, which has to call through to Vulkan, which then returns the geometry.

And if the Vulkan driver performs a display server roundtrip to get that geometry, it becomes very slow at extreme framerates.

The solution here is obvious: make Mesa's WSI avoid roundtrips when polling geometry. This doesn't fix non-Mesa drivers, but it does provide a substantial (~10%) boost to performance for the ones affected.

## Brass Tacks: Presentation
Another big difference between Gallium DRI and VK WSI is present handling.

In DRI, presentation happens.

In VK WSI, the event queue is flushed, and then presentation happens, then the result of presentation is waited on and checked, and the value is returned to the app.

It's a subtle difference, but I'm sure my genius-level readers can pick up on it.

The reasoning for the discrepancy is clear: [vkQueuePresentKHR](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkQueuePresentKHR.html) has a return value, which means it *can* return values to indicate various failure/suboptimal modes. But is this a good idea?

Yes, but also no. On one hand, certain returns are important so the app can determine when to reallocate the swapchain or make other, related changes. On the other hand, doing all this extra work impacts WSI performance by a substantial amount, and none of it is truly useful to applications since the same values can be returned by acquire.

So I deleted it.

That was a ~60% perf boost.

## Brass Tacks: Reducing Redundant WSI Operations
The final frontier of WSI perf is syncfile operations. TL;DR Vulkan WSI is explicit, which means there's a file descriptor fence passed back and forth between kernel and application to signal when a swapchain image can be used/reused, and all of these operations have costs. These costs aren't typically visible to applications, but when 4000ns can be a 10% performance boost, they start to become noticeable.

Now one thing to note here is that the DRI frontend uses implicit sync, which means drivers are allowed to do some tricks to avoid extra operations related to swapchain fencing. VK WSI, however, uses explicit sync, which means all these tricks are ~~still totally legal~~NOT AT ALL LEGAL. This means that, in a sense, VK WSI will always be slightly slower. Probably less than 10,000ns per frame, but, again, this is significant when running at extreme framerates.

The only operation I could sensibly eliminate from the presentation path was the dmabuf semaphore export. Semaphores are automatically reset after they are signaled, so this means there's no need to continually export the same semaphore for every present.

[![honest-work.png]({{site.url}}/assets/honest-work.png)]({{site.url}}/assets/honest-work.png)

## Brass Tacks: Also Swapchain Size
The DRI frontend uses a swapchain size of four images when using IMMEDIATE and the swap mode returned by the server is 'flip'.

VK WSI always uses three images.

Switching to four swapchain images for this exe uncapped the perf a bit further.

# Results
Nobody's expecting any huge gains here, but that's obviously a lie since the only reason anyone comes to SGC is for the huge gains.

And huge gains there are.

My new glmark2 results:
* **RadeonSI** - 25,318fps
* **Zink** - 16,998fps

Still nowhere near caught up, but this cuts roughly 40,000ns (~40%) off zink's frametime, which is big enough to qualify for a mention.

# Unethical Gains
These changes are still a bit experimental. I've tested them locally and gotten good results without issues in all scenarios, but that doesn't mean they're "correct". VK WSI is used by a lot of applications in a lot of ways, so it's possible there's scenarios I haven't considered.

If you're interested in testing, the [MR is here](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/23835).
