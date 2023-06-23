---
published: false
---
# When perf Is Too Slow

Keen-eyed readers of the blog will have noticed over the past couple weeks will have noticed an obvious omission in my furious benchmark-related handwaving.

I didn't mention glmark2.

[![vulkan.svgz](https://phoronix.com/benchmark/result/radeonsi-vs-zink-opengl-benchmarks-2023/glmark2-1920-x-1080.svgz)](https://phoronix.com/benchmark/result/radeonsi-vs-zink-opengl-benchmarks-2023/glmark2-1920-x-1080.svgz)

Yes, that glmark2.

Except my results were a bit different.

The thing about glmark2 is it's just a CPU benchmark of drivers at this point. None of the rendering is in any way intensive, GPUs have gotten far more powerful than they were at the time the thing was written, and CPUs are now pushing tens of thousands of frames per second.

Yes, tens of thousands.

I set out to investigate this using the case with the biggest perf delta, which was `build:use-vbo=true`:
* **RadeonSI** - 23,000fps
* **Zink** - 10,000fps

Brutal.

Naturally I went to my standby for profiling `perf`. I found some stuff, and I fixed it all, and surprise surprise the numbers didn't change. Clearly there was some blocking going on that was affecting perf. Could the flamegraph tools for `perf` highlight this issue for me?

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

This is over 400 times faster.

