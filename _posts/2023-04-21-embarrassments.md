---
published: false
---
## This Week

I escaped my RADV pen once again. I know, it's been a while, but every so often [my handler](https://twitter.com/Plagman2) gets distracted and I sprint for the fence with all my might.

This time I decided to try out a shiny new Intel Arc A770 that was left by my food trough.

The Dota2 performance? Surprisingly good. I was getting 100+ FPS in all the places I expected to have good perf and 4-5 FPS in all the places I expected to have bad perf.

The GL performace? Also surprisingly good\*. Some quick screenshots:

DOOM 2016 on Iris:

[![doom-iris-dg2.png]({{site.url}}/assets/doom-iris-dg2.png)]({{site.url}}/assets/doom-iris-dg2.png)

Playable.

And of course, this being a zink blog, I have to do this:

[![doom-zink-dg2.png]({{site.url}}/assets/doom-zink-dg2.png)]({{site.url}}/assets/doom-zink-dg2.png)

The perf on zink is so good that the game thinks it's running on an NVIDIA GPU.

If anyone out there happens to be a prominent hardware benchmarking site, this would probably be an interesting comparison on the upcoming Mesa 23.1 release.

## More News
I've seen a [lot of people](https://github.com/ValveSoftware/steam-for-linux/issues/9298#issuecomment-1483846775) with AMD hardware getting hit by the [Fedora 38 / LLVM 16 update crash](https://github.com/ValveSoftware/Dota-2/issues/2285#issuecomment-1502616760). While this is unfortunate, and there's nothing that I, a simple meme connoisseur, can do about it, I have some simple workarounds that will enable you to play all your favorite games without issue:
* For Vulkan-based games, delete/rename your Lavapipe ICD (`rm /usr/share/vulkan/icd.d/lvp_icd.*`)
  * This will be workaroundeded in the upcoming Mesa releases and is probably [already](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/22600) fixed in main
* For OpenGL-based games, set `MESA_LOADER_DRIVER_OVERRIDE=zink %command%` in your game's launch options

I realize the latter suggestion seems meme-adjacent, but so long as you're on Mesa 23.1-rc2 or a recent git build, I doubt you'll notice the difference for [most games](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8223).

You can't run a full desktop on zink yet, but you can now play most games at native-ish performance. Or better!

## Lastly
I've made mention of Big Triangle a number of times on the blog. Everyone's had a good chuckle, and we're all friends so we know it's an inside joke.

But what if I told you I was serious each and every time I said it?

What if Big Triangle really does exist?

I know what you're thinking: Mike, you're not gonna get me again. You can't trick me this time. I've seen this coming for—

**CHECK OUT THESE AMDVLK RELEASE NOTES!**

[![amdvlk1.png]({{site.url}}/assets/amdvlk1.png)]({{site.url}}/assets/amdvlk1.png)

Incredible, they're finally supporting that one extension [I've been saying]({{site.url}}/sp33d2) is crucial for having good performance. Isn't that ama—

[![amdvlk2.png]({{site.url}}/assets/amdvlk2.png)]({{site.url}}/assets/amdvlk2.png)

And they've even added an app profile for zink! I assume they're going to be slowly rolling out all the features zink needs in a controlled manner since zink is a known good-citizen when it comes to behaving within the boundaries of—

[![amdvlk3.png]({{site.url}}/assets/amdvlk3.png)]({{site.url}}/assets/amdvlk3.png)

...

[![7.png]({{site.url}}/assets/clown/7.png)]({{site.url}}/assets/clown/7.png)