---
published: true
---
## A quick break
I've been blogging pretty in-depth about Zink and related code for a while, so let's do a quick roundup with some future blog post spoilers.

**I previously talked about these, now they're merged:**
* [Yesterday's ANV MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5792) merged
* [ARB_depth_clamp](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5495)
* [gl_FragColor fixup](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5687)

**Done but awaiting dependencies before merge:**
* [UBO support](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2872) from last week's involved blog series
* [Primitive restart](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2873) which I'll likely cover in a quick post tomorrow
* [GLSL 1.40](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2874) was a variety of misc fixups
* [GL 3.1](https://gitlab.freedesktop.org/mesa/mesa/-/issues?milestone_title=Zink+OpenGL+3.1+support) depends on the above items, so that's done
* [Geometry shaders](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3175) are working about as well as they can after tackling that over the past day or so
* GLSL 1.50 is at 98% of piglit tests passing (this can't reach 100% due to architecture limitations)
* GL 3.2 doesn't have a milestone, but it's done (depth clamp, geometry shaders, and GLSL 1.50 were the last remaining items)
* [GL 3.3](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3246) needs 3 extensions and the GLSL bump


Also, Antonio Caggiano has dipped a toe into Zink-land and is investigating fixing up some issues we have with depth/stencil buffer operations!

## Goals
It's been over a month of daily posts, so again, here are 3 goals for this blog for any recently-joined readers:
* Document things I/others are doing in Zink and Zink-affecting components\
This is coincidentally the only documentation for some of the mesa APIs that I'm blogging about.
* Show readers who maybe have strong coding/graphics backgrounds but haven't done driver development that drivers aren't some kind of black box code that's *too hard* for hobbyists to jump into
* Also possibly describe some problem-solving strategies and implementation handling that might be useful
