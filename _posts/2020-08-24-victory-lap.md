---
published: true
---
## After Some Time

```
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: Collabora Ltd (0x8086)
    Device: zink (Intel(R) Iris(R) Plus Graphics (ICL GT2)) (0x8a52)
    Version: 20.3.0
    Accelerated: yes
    Video memory: 11690MB
    Unified memory: yes
    Preferred profile: core (0x1)
    Max core profile version: 4.6
    Max compat profile version: 3.0
    Max GLES1 profile version: 1.1
    Max GLES[23] profile version: 3.2
OpenGL vendor string: Collabora Ltd
OpenGL renderer string: zink (Intel(R) Iris(R) Plus Graphics (ICL GT2))
OpenGL core profile version string: 4.6 (Core Profile) Mesa 20.3.0-devel (git-b229c29e4d)
OpenGL core profile shading language version string: 4.60
```

It's been about three months since I jumped into the project to learn more about graphics drivers, and zink has now gone from supporting GL 3.0 to GL 4.6 and GLES 3.2 compatibility*. Currently I'm at a [91% pass rate on piglit tests]({{site.url}}/assets/new3.tbz2) while forcing ANV to do unsupported fp64 ops, which seems like a pretty good result.

**in my branch*

Not bad for some random guy sitting in his basement.

## What's Next?
Well, first up is probably going to be a bit of a break. I've been surfing the gfx pipeline nonstop for the past couple weeks, as has been hinted at by my lack of posts here, and so there's some non-graphics stuff I've been putting off doing.

After that, I imagine I'll be going back to work on some of these test failures and go through more of the CTS results since there's a lot of test coverage over there.

There's also lots of [refactoring work](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3447) that I've [flagged](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3361) for [doing](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3359) at [some point](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3284) when I [had time](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3404) or interest.

Also, more blogging.

## When Will This Be Merged?
Patch review is a complex process during which code must be carefully examined for defects. It takes time to do it right, and it shouldn't be rushed.

With that said, I've got around 300 patches in my branch (likely far more once I break up some of the giant wip ones), so it's unlikely that all of it will be landed anytime in the very near future.

Anyone interested in trying things out immediately can pull my [zink-wip](https://gitlab.freedesktop.org/zmike/mesa/-/commits/zink-wip) branch and follow the [wiki instructions](https://gitlab.freedesktop.org/kusma/mesa/-/wikis/zink-building-and-running) for building and running the zink driver on top of existing distro-provided vulkan drivers without any other changes, potentially even submitting bug reports for me to look at.


Tomorrow, I'll be starting a dive here into the tessellation shader implementation and some "cool" things I had to do there.
