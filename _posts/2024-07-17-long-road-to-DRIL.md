---
layout: post
title: Long Road To DRIL
published: true
---
# I'm Cookin

Lot of stuff happening. I can't talk about much of it yet, but trust me when I say the following:

It's happening.

When it happens, you'll know what I meant.

# Today Is A Great Day

Remember way back when I [put DRI interfaces on notice]({{site.url}}/post-interfaces/)?

Now, only four months later, DRI interfaces are finally [going away](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28378).

Begun by @ajax and then finished off by me and Pavel (ghostwritten by @daniels), the DRIL (DRI Legacy) interface is a tiny shim which matches Xorg's ABI expectations to provide a list of sensible fbconfig formats during startup. Then it does nothing. And by doing nothing, it saves the rest of Mesa from being shackled to ancient ABI constraints.

Let the refactoring begin.

# But Wait, There's More!

Obviously I'm not going to stop here. SGC leaves no code half-krangled. That's why, as soon as DRIL lands, I'll also be hammering in [this followup MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/29771) which finally makes all the GL frontends link directly to the Gallium backend driver.

Why is this so momentous, you ask? How many of you have gotten the error `DRI driver not from this Mesa build` when trying to use your custom Mesa build?

With this MR, that error is going away. Permanently. Now you can have as many Mesa builds on your system as you want. No longer do you need to set `LIBGL_DRIVERS_PATH` for any reason.

The future is here.
