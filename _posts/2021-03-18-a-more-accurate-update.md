---
published: true
---
## Where We At

The blogging is happening once again, and it feels good.

I did a brief recap a couple posts ago, but I've gotten some sleep since then, and now it's time for the real deal. Here's what you missed while you were asleep and not following the [Mesa RSS feed](https://cgit.freedesktop.org/mesa/mesa/atom/?h=master) like all the coolest people I know:
* zink-wip is shrinking. no, really. it's down from close to 600 patches to around 200 after the past month
* descriptor caching has landed, likely yielding around an 80-100% performance increase across the board for applications which were CPU-bound. huge, huge thanks to legendary reviewer, RADV developer, and all around great guy, Bas Nieuwenhuizen for tackling this [way-too-fucking-big, 50+ patch MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/9348) in an amazingly short amount of time
* no, zink still can't render glxgears properly
* zink now has lavapipe-based CI jobs going to prevent regressions!
* Erik Faye-Lund has managed a [one-liner MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/9649) that fixed over a hundred unit tests in CI
* buffer invalidation is in, which lets zink avoid stalling the GPU in a lot of cases, also improving performance
* with all this said, another giant thanks to l\*v\*pipe guru and part-time motivational graphics coach, Dave Airlie, who has reviewed nearly 200 of my patches in this release cycle
* lavapipe is ramping up towards Vulkan 1.1 support, which will, other than being a great milestone and feat of software engineering, enable zink CI testing through GL 4.5
* less than a hundred patches to go before zink's threaded context support is up for merging

Good times.
