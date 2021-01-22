---
published: false
---
## Return of the Blog

I meant to blog a couple times this week, but I kept getting sidetracked by various matters. Here's a very brief recap on what's happened in zinkland over the past week.
# Important MRs
* [basic atomic counter support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8330)
* [texture clear support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8512)
* [enhanced layout support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8515)
* [RADV implicit flush hacks](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8284)
* [crash fix for GALLIUM_HUD users](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8564)
* [codegen improvements lowering the bar for feature usage on various VK drivers](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8521)
* [shader image support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8504)
* [GLSL 420 and GL 4.2 support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8620)
* [vertex stride fixups for TORCS racing game](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8588)

And a bunch more extensions related to GL 4.3+ are now enabled.

A new `zink-wip` snapshot is finally up after most of a week spent fighting regressions. Anyone updating from a previous (e.g., 20201230) snapshot will find:
* a ton of garbage patches which haven't been properly split or pruned in any way and are likely unreadable/unbisectable
* a new descriptor manager which doesn't cache and instead uses templates and incremental updates to provide comparable performance to the caching one; `ZINK_CACHE_DESCRIPTORS=1` will use the caching version
* tons of optimizations for reducing driver overhead
* a prototype for a new Vulkan extension providing direct multidraw functionality (as well as accompanying implementations for both ANV and RADV) which very slightly improves performance
* lots of corner case bug fixes for things much earlier in the branch

All told, zink should now be (slightly, possibly even imperceptibly) faster as well as being even less bug-prone.

I'm pretty beat after this week, so that's all for now. Hoping to return to more normal, in-depth coverage of driver internals next week.