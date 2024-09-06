# What Am I Even Doing

It was [some time ago](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/972) that I created my first MR touching WSI stuff. 

That was also the first time I broke Mesa.

Did I learn anything?

The answer is no, but then again it would have to be given the topic of this sleep-deprived post.

# Maybe I'm The Problem

WSI has a lot of issues, but most of them stem from its mere existence. If people stopped wanting to see the triangles, we would all have easier lives and performance would go through the fucking roof. That's ignoring the raw sweat and verbiage dedicated to insane ideas like [determining the precise time at which the triangles should be made visible on a display](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/248) or [literally how are colors](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/14).

I'm nowhere near as smart as the people arguing about these things: I'm the guy who plays jenga with the tower constructed from popsicle sticks, marshmallow fluff, and wishful thinking. That's why, a while ago, I [declared war]({{site.url}}/post-interfaces/) on DRI interfaces and then also [definitely won that war]({{site.url}}/long-road-to-DRIL) [without](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11807) [any](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11827) [issues](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11809). [In](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11808) [fact](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11682)[,](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11836) [it](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11545) [works](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11546)[ ](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11560)[perfectly](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11544).

[But](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30311) [why](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30376) [did](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30454) [I](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30449) [embark](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30328) [upon](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30485) [this](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30556) [journey](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30588) [which](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30851) [required](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30426) [absolutely](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30951) [no](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31013) [fixups](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30979)[?](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30964)

The answer lies in architecture. In the before-times, DRI (a massively overloaded acronym that no longer means anything) allowed Xorg to plug directly into Mesa to utilize hardware-accelerated rendering. It sidestepped the GL API in favor of a contract with Mesa that certain API would never change. And that was great for Xorg since it provided an optimal path to do xserver stuff. But it was (eventually) terrible for Mesa.

# Renegotiation

When an API contract is made, it remains binding forever. A case when the contract is broken is called a Bug Report. Mesa has no bugs, however, except for the ones I didn't cause, and so this DRI contract that enables Xorg to shortcut more sensible APIs like EGL remains identical to this day, decades later. What is not identical, however, is Mesa.

In those intervening years, Mesa has developed into an entire ecosystem for driver development and other, [less sane ideas]({{site.url}}/CLosure). Gallium was created and then became the only method for implementing GL drivers. EGL and GBM are things now. But still, that DRI contract remains binding. Xorg must work. Like that one reviewer who will suggest changes for every minuscule flaw in your stupid, idiotic, uneducated, cretinous abuse of whitespace, it is not going away.

DRIL was the method by which Mesa could finally unshackle itself. The only parts of DRI still used by Xorg are for determining rendertarget capabilities, effectively `eglGetConfigs`. So [@ajax](https://googlethatforyou.com?q=the%20pub) and I punted out the relevant API into a stub which is mostly just a wrapper around `eglGetConfigs`. This enabled change and cleanup in every part of the codebase that was previously immutable.

# Bidirectional Hell

As anyone who has tried to debug Mesa's DRI frontend knows, it sucks. It's one of the worst pieces of code to debug. A significant reason for this is (was) how the DRI callback system perpetuated circular architecture.

At the time of DRIL's merge, a user of GLX/EGL/GBM would engage with this sort of control flow:
1. GLX/EGL/GBM API call
2. direct API internals
3. function pointer into `gallium/frontends/dri`
4. DRI frontend
5. function pointer back to GLX/EGL/GBM
6. <loop back to 2 until operation completes>
99. return to user

In terms of functionality, it was functional. But debugging at a glance was impossible, and trying to eyeball any execution path required the type of PhD held by fewer than five people globally. The cyclical back-and-forth function pointering was a vertical cliff of a learning curve for anyone who didn't already know how things worked, and even things as "simple" as `eglInitialize` went through several impenetrable cycles of idiot-looping to determine success or failure. The absolute state of it made adding new features a nightmarish and daunting prospect, and reviewing any changes had, at best, even odds of breaking things because of how difficult it is to test this stuff.

# Better Now?

Maybe.

The juiciest refactoring is over, and now function pointering only occurs when the DRI frontend needs to access API-specific data for its drawables. It's actually possible to follow execution just by reading the code. Not that it's necessarily easy, but it's possible.

There's still a lot of work to be done here. There's still some corner case bugs with DRIL, there's probably EGL issues that have yet to be discovered because much of that code is still fairly opaque, and half the codebase is still prefixed with `dri2_`.

At the least, I think it's now possible to work on WSI in Mesa and have some idea what's going on. Or maybe I've just been down in the abyss for so long that I'm the one staring back.

# Onward

I've been cooking. I mean like [*really* cooking](https://www.youtube.com/watch?v=QrGrOK8oZG8). Expect big things related to the number **3** later this month.

\* UPDATE: At the urging of my legal team, I've been advised to mention that no part of this post, blog, or site has any association with, bearing on, or endorsement from **Half Life 3**.