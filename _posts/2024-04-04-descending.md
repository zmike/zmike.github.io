# Into The Spiral of Madness

I know what you're all thinking: there have not been enough blog posts this year. As always, my highly intelligent readers are right, and as always, you're just gonna have to live with that because I'm not changing the way anything works. SGC happens when it happens.

And today. As it snows in April. SGC. Is. Happening.

Let's begin.

# In The Beginning, A Favor Was Asked
I was sitting at my battlestation doing some very ordinary **REDACTED** work for **REDACTED** and friend of the blog, Samuel "Shader Objects" Pitoiset (he has legally changed his name, please be respectful), came to me with a simple request. He wanted to be able to enable [VK_EXT_shader_object](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_shader_object.html) for the radv-zink jobs in mesa CI as the final part of his year-long bringup for the extension. This meant that all the tests passing without shader objects needed to also pass with shader objects.

This should've been easy; it was over a year ago that the Khronos blog famously and confusingly [announced](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today) that pipelines were dead and nobody should ever use them again (paraphrased). A year is more than enough time for everyone to collectively get their shit together. Or so you might think.

Turns out shader objects are hard. This simple ask sent me down a rabbithole the likes of which I had never imagined.

It started normally enough. There were a few zink tests which failed when shader objects were enabled. Nobody was surprised; I wrote the zink usage before validation support had landed and also before anything but lavapipe supported it. As everyone is well aware, lavapipe is the best and most handsome Vulkan driver, and just by using it you eliminate all bugs that your application may have. RADV is not, and so there are bugs.

A number of them were simple:
* [invalid location assignment for patch variables](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10414)
* harmless [VVL spam](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10457) about missing dynamic state setters
* Vulkan spec had [broken corner cases](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10737)
* [reporting wrong value](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10750) for max patch components
* also patch variables were differently broken in [another way](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28296)

The list goes on, and longtime followers of the blog are nodding to themselves as they skim the issues, confirming that they would have applied all the same one-liner fixes.

Then it started to get crazy.

# Locations, How Do They Work?
I'm a genius, so obviously I know how this all works. That's why I'm writing this blog. Right?

[![smart-or-blogger.png]({{site.url}}/assets/smart-or-blogger.png)]({{site.url}}/assets/smart-or-blogger.png)

Right. Good. So Samuel comes to me, and he hits me with [this absolute brainbuster](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10809) of an issue. An issue so tough that I have to perform an internet search to find a credible authority on the topic. I found this amazing and informative [site]({{site.url}}/adventures-in-linking) that exactly described the issue Samuel had posted. I followed the staggering intellect of the formidable author and blah blah blah yeah obviously the only person I'd find writing about an issue I have to solve is past-me who was too fucking lazy to actually solve it.

