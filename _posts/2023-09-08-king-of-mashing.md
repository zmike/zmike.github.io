---
published: true
---
# Overdue

It's been a busy week, and I've got posts I've been meaning to write. The problem is they're long and I'm busy. That's why I'm doing a shorter post today, since even just getting this one out somehow took 4+ hours while I was continually sidetracked by "real work".

But despite this being a shorter post, don't worry: the memes won't be shorter.

# Fighting!

I got a ticket [very recently](https://gitlab.freedesktop.org/mesa/mesa/-/issues/9119) that I immediately jumped on and didn't at all procrastinate or forget about. The ticket concerned a little game called **THE KING OF FIGHTERS XIV**.

Now for those of you unfamiliar, [The King of Fighters](https://en.wikipedia.org/wiki/The_King_of_Fighters) is a long-running fighting game franchise which debuted in the 90s. At the *arcade*. Pretty sure I played it once. But at like a retro arcade or something because I'm not that old, fellow kids.

The bug in question was that when a match is won using a special move, half the frame would misrender:

[![kof.jpg](https://gitlab.freedesktop.org/mesa/mesa/uploads/18bc059de940f2f0c090af6221a62574/kofiv_zink.jpg)](https://gitlab.freedesktop.org/mesa/mesa/uploads/18bc059de940f2f0c090af6221a62574/kofiv_zink.jpg)

Heroically, the reporter posted a number of apitrace captures. Unfortunately that effort ended up being ineffectual since it did nothing but reveal [yet another apitrace bug](https://github.com/apitrace/apitrace/issues/892) related to VAO uploads which caused replays of the traces to crash.

It was the worst kind of bug.

I was going to have to ~~play a game~~test the defect myself.

It would prove to be the greatest test of my skill yet. I would have to:
* start the correct game
* win a match
* win a match using a special move
* renderdoc capture at the exact right time during the finishing animation to capture the bug

Was I successful?

I'm not saying there's someone out there who's worse at ~~the game~~the test app than a guy ~~playing~~performing exploratory tests on his keyboard under renderdoc. That's not what I'm saying.

[![kof.png]({{site.url}}/assets/kof.png)]({{site.url}}/assets/kof.png)

# Eddie Gordo Got Mad At This Combo

The debug process for this issue was, in contrast to the capture process, much simpler. I attribute this to the fact that, while I don't own a gamepad for use with whatever test apps need to be run, I do have a code controller that I use for all my debugging:

[![code-controller.png]({{site.url}}/assets/code-controller.png)]({{site.url}}/assets/code-controller.png)

I've been hesitant to share such pro strats on the blog before, but SGC has been around for long enough now that even when the copycats start vlogging about my tech and showing off the frame data, everyone will recognize where it came from. All I ask is that you post clips of tournament popoffs.

Using my code controller, I was able to perform a `debug -> light code -> light code -> debug -> heavy code -> compile -> block -> post meme -> reboot -> heavy code -> heavy code` combo for an easy W.

To break down this advanced sequence, a small `debug` reveals that the issue is a render area clamped to 1024x1024 on a 1920x1080 frame. Since I have every line of the codebase memorized (zink main don't @ me) it was instantly obvious that some poking was in order.

Vulkan has this pair of (awful) VUs:

```
VUID-VkRenderingInfo-pNext-06079
If the pNext chain does not contain VkDeviceGroupRenderPassBeginInfo or its deviceRenderAreaCount member is equal to 0, the width of the imageView member of any element of pColorAttachments, pDepthAttachment, or pStencilAttachment that is not VK_NULL_HANDLE must be greater than or equal to renderArea.offset.x + renderArea.extent.width

VUID-VkRenderingInfo-pNext-06080
If the pNext chain does not contain VkDeviceGroupRenderPassBeginInfo or its deviceRenderAreaCount member is equal to 0, the height of the imageView member of any element of pColorAttachments, pDepthAttachment, or pStencilAttachment that is not VK_NULL_HANDLE must be greater than or equal to renderArea.offset.y + renderArea.extent.height
```

which don't match up at all to GL's ability to throw whatever size framebuffer attachments at the GPU and have things come out fine. A long time ago I wrote [this MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16947) to clamp framebuffer size to the smallest attachment. But in this particular case, there are three framebuffer attachments:
* 1920x1080
* UNUSED
* 1920x1080

The unused attachment ends up clamping the framebuffer to a smaller region to avoid violating spec, and this breaks rendering. Some `light code` pokes to skip clamping for NULL attachments open up the combo. Another quick `debug` doesn't show the issue as being resolved, which means it's time for some `heavy code`: checking for unused attachments in the fragment shader during renderpass start.

Naturally this triggers a full tree `compile`, which is a `block`ing operation that gives me enough time to execute a `post meme` for style points. The downside is that I'm using an AMD system, so as soon as I try to run the new code it hangs--it's at this point that I nail a `reboot` to launch it into orbit.

I'm not looking for a record-setting juggle, so I finish off my combo with a `heavy code -> heavy code` finisher to hack in attachment write tracking for TC renderpass optimization and then [plumb it through the rest of my stack](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25121) so unused attachments will skip all renderpass-related operations.

Problem solved, and all without having to personally play any games.

# Next Week
I'll finally post that one post I've been planning to post for weeks but it's hard and I just blew my entire meme budget for the month today so what is even going to happen who knows.
