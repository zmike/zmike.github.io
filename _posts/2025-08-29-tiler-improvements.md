# Super Late Code

Meant to blog about this last quarter, but somehow another two months went by and here we are.

A while back, I did some work to improve zink performance on tiling GPUs. Namely this entailed adding renderpass tracking into threaded-context, and also implementing command stream reordering, and inlining swapchain resolves, and framebuffer discards, and actually maybe it's more than just "some" work. All of this amounted to improved performance by reducing memory bandwidth.

How much improved performance? All of it.

And then, around two months ago, a colleague told me he was no longer going to use zink on his tiling GPU.

# Devastated

Some of you noticed that the blog has gone quiet in recent times. I'm going to take this opportunity to foist all the blame onto that colleague: to preserve his identity, let's just call him Gabe.

Gabe came to me a few months ago and told me zink was too slow. Vulkan was better. Faster. More "reliable".

I said there's no way that could be true; I've put way more bugs into Vulkan than I have into zink.

Unblinking, he stared at me across the digital divide. I task-switched to important whitespace cleanups.

Time passed, and I pulled myself together. I compiled some app traces. Analyzed them. Did some deep thinking. There was one place where zink indeed could be less performant than this "Vulkan" thing. The final frontier of driver performance. Some call it graphics heaven.

I call it hell.

# Web Browsers

Chrome is the web browser, and, statistically, everyone uses it. It ships on desktops and phones, embeds in apps, and even allows you to read this blog. Haters will say *No I uSe FiReFoX*, but they may as well be Netscape users in the year 2000.

In the past, Chrome defaulted to using GL, which made testing easy. Now, however, `--disable-features=Vulkan` is needed to return to the comfort of an API so reliable it no longer receives versioned updates. Looking at an apitrace of Chrome, I saw a disturbing rendering pattern that went something like this:

* draw some element on a page using multisampled FBO1
* resolve FBO1 to texture1
* composite texture1 onto larger FBO2/texture2
* composite texture2 onto even larger, multisampled FBO3
* resolve FBO3 to swapchain
* present

In this case, zink would correctly inline the FBO3/swapchain resolve at the end, but the intermediate multisampled rendering on `FBO1` would pay the full performance penalty of storing the multisampled image data and then loading it again for the separate resolve operation.

I'd like to say it was simple to inline this intermediate resolve. That I just slapped [a single MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/35477) into mesa and it magically worked. Unfortunately, nothing is ever that simple. There were [minor](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/35777) [fixups](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/36069) [all over](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/36521) [the place](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/36576). And this brought me to the real insanity.

Chrome has bugs too.

# Literal Hell

Let's take a concrete example: launch Chrome with `--disable-features=Vulkan` and check out this tiny SVG: [chromebug.html]({{site.url}}/assets/chromebug.html)

This is most likely what you see:

[![chromebug-good.png]({{site.url}}/assets/chromebug-good.png)]({{site.url}}/assets/chromebug-good.png)

The reason you see this is because you are on a big, strong desktop GPU which doesn't give a shit about load/store ops or uninitialized GPU memory. You're driving a giant industrial bulldozer on your morning commute: traffic no longer exists and stop signals are fully optional. On a wimpy tiling GPU, however, things are different.

Using a recent version of zink, even on a desktop GPU, you can run the same Chrome browser using `ZINK_DEBUG=rp,rploads` to enable the same codepaths used by tilers and also clear all uninitialized memory to red. Now load the same SVG, and you'll see this:

[![chromebug-bad.png]({{site.url}}/assets/chromebug-bad.png)]({{site.url}}/assets/chromebug-bad.png)

 It took nearly a week of pair debugging and a new zink debug mode to prune down test cases and figure out what was happening. All around the composited SVG texture, memory is uninitialized.

 But this only shows up on tiling GPUs. And only if the driver is doing near-lethal amounts of very legal renderpass optimizations.

 This fast is too fast.
