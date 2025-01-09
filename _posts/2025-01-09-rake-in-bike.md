# First Perf of the Year

I got a ticket last year about this game [Everspace](https://gitlab.freedesktop.org/mesa/mesa/-/issues/12063) having bad perf on zink. I looked at it a little then, but it was the end of the year and I was busy doing other stuff. More important stuff. I definitely wasn't just procrastinating.

In any case, I didn't fix it last year, so I dusted it off the other day and got down to business. Unsurprisingly, it was still slow.

# Easing Into Speed

The first step is always a flamegraph, and as expected, I got a hit:

[![queryperf.png]({{site.url}}/assets/everspace/queryperf.png)]({{site.url}}/everspace/queryperf.png)

Huge bottlenecking when checking query results, specifically in semaphore waits. What's going on here?

What's going on is this game is blocking on timestamp queries, and the overhead of doing `vkWaitSemaphores(t=0)` to check drm syncobj progress for the result is colossal. Who could have guessed that using core Vulkan mechanics in a hotpath would obliterate perf?

Fixing this is very stupid: directly checking query results with `vkGetQueryPoolResults` avoids syncobj access inside drivers by accessing what are effectively userspace fences, which Vulkan doesn't directly permit. If an app starts polling on query results, zink [now](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31823) uses this rather than its usual internal QBO mechanism.

Bottleneck uncorked and performance fixed. Right?

# Naaaaaa

The perf is still pretty bad. It's time to check in with the doctor. Looking through some of the renderpasses reveals all kinds of begin/end tomfoolery. Paring this down, renderpasses are being split for layout changes to toggle feedback loops:

[![feedback.png]({{site.url}}/assets/everspace/feedback.png)]({{site.url}}/everspace/feedback.png)

The game is rendering to one miplevel of a framebuffer attachment while sampling from another miplevel of the same image. This breaks zink's heuristic for detecting implicit feedback loops. [Improvements here](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/32950) tighten up that detection to flatten out the renderpasses.

# Gottagofastium

Perf recovered: the game runs roughly 150% faster, putting it on par with RadeonSI. Maybe some other games will be affected? Who can say.