---
published: true
---
## Fast Friday

In short, [an issue was filed recently](https://gitlab.freedesktop.org/mesa/mesa/-/issues/4935) about getting the Nine state tracker working with zink.

Was it the first? [No.](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3170).

Was it the first one this year? Yes.

Thus began a minutes-long, helter-skelter sequence of events to get Nine up and running, spread out over the course of a day or two. In need of a skilled finagler knowledgeable in the mysterium of Gallium state trackers, I contacted the only developer I know with a rockstar name, Axel Davy. We set out at dawn, and I strapped on my parachute. It was almost immediately that I heard a familiar call: [there's a build issue](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11467).

Next stop was crashing in [unimplemented interface methods](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11466) with a stopover in [flailing about wildly in TGSI what even is this](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11468) before I arrivated at my target:

[![nine.png]({{site.url}}/assets/nine.png)]({{site.url}}/assets/nine.png)

Ah, glorious triangles.
