---
published: false
---
# EBUSY

As everyone knows, SGC goes into yearly hibernation beginning in November. Leading up to that point has been a mad scramble to nail down all the things, leaving less time for posts here.

But there have been updates, and I'm gonna round 'em all up.

# R A Y T R A C W T F
Friend of the blog and future Graphics scientist with a PhD in WTF, Konstantin Seurer, has been hard at work over the past several weeks. Remember earlier this year when he implemented `VK_EXT_descriptor_indexing` for Lavapipe? Well he's at it again, and this time he's aimed for something bigger.

He's now [implemented raytracing](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25616) for Lavapipe.

It's a tremendous feat, one that sets him apart from the other developers who have not implemented raytracing for a software implementation of Vulkan.

# CLosure
I blogged (or maybe imagined blogging) about RustiCL progress on zink last year at XDC, specifically the time renowned pubmaster Karol Herbst handcuffed himself to me and refused to divulge the location of the key (disguised as a USB thumb drive in his laptop) until we had basic CL support functioning. That time is finally turning into something useful as CL support for zink will soon be [merged](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24839).

While I can't reveal too much about the performance as of yet, what I can say now is that it's roughly 866% faster.

# Fixups
A number of longstanding bugs have recently been fixed.

## Wolfenstein Face
Anyone who has tried to play one of the modern Wolfenstein GL games on RADV has probably seen this abomination:

[![wolf-face.png]({{site.url}}/assets/wolf-face.png)]({{site.url}}/assets/wolf-face.png)

Wolfenstein Face affects a very small number of apps. Actually just the Wolfenstein (The New Order / The Old Blood) games. I'd had a [ticket](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8988) open about it for a while, and it turns out that this is a [known issue](https://gitlab.freedesktop.org/mesa/mesa/-/issues/5753) in D3D games which has its own workaround. The workaround is now going to be [applied for zink](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25642) as well, which should resolve the issue.

## Apitrace: The Final Frontiers

Since the dawn of time, experts have tried to obtain traces from games with rendering bugs, but some of these games have historically been resistant to tracing.

* A number of games could be traced, but then replaying those traces would crash at a certain point. This is now [fixed](https://github.com/apitrace/apitrace/pull/899), enabling better bug reporting for a large number of AAA games from the the last decade.
* Another set of games using the id Engine could record traces, but then replaying them would fail to render correctly:

[![wolf-trace.png]({{site.url}}/assets/wolf-trace.png)]({{site.url}}/assets/wolf-trace.png)

This affects (at least) `Wolfenstein: The Old Blood` and `DOOM2016`, but the problem has been identified, and a fix is on the way.