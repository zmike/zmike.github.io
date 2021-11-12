---
published: false
---
## A Long Time Coming

Zink can now run all display platform flavors of Weston (and possibly other compositors?). Expect it in zink-wip later today once it passes another round of my local CI.

Here it is in DRM:

[![wayland-screenshot-2021-11-12_13-25-25.png]({{site.url}}/assets/wayland-screenshot-2021-11-12_13-25-25.png)]({{site.url}}/assets/wayland-screenshot-2021-11-12_13-25-25.png)

## Under Construction
This has a lot of rough edges, mostly as it relates to X11. In particular:
* xservers (including xwayland) can't run because GLAMOR is hard
* some apps (e.g., Unigine Heaven) randomly get killed by the xserver for unknown reasons
* if you're very lucky, you can hit [a Vulkan WSI deadlock](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/13564)

## How?
I'd go into details on this, but honestly it's going to be like a week of posts to detail the sheer amount of chainsawing that's gone into the project.

Stay tuned for that and more next week.