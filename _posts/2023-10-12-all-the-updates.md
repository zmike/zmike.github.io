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

## Apitrace: The Final Frontier

Since the dawn of time, experts have tried to obtain traces from games with rendering bugs, but some of these games have historically been resistant to tracing.

* A number of games could be traced, but then replaying those traces would crash at a certain point. This is now [fixed](https://github.com/apitrace/apitrace/pull/899), enabling better bug reporting for a large number of AAA games from the the last decade.
* Another set of games using the id Engine could record traces, but then replaying them would fail to render correctly:

[![wolf-trace.png]({{site.url}}/assets/wolf-trace.png)]({{site.url}}/assets/wolf-trace.png)

This affects (at least) `Wolfenstein: The Old Blood` and `DOOM2016`, but the problem has been identified, and a fix is on the way.

# The Real Post
Any other, lesser blogger would've saved this for another post in order to maximize that posting frequency metric, but here at SGC the readers get a full meal with every post. Since I'm not going to XDC this year, consider this the thing I might have given a presentation on.

During my executive senior keynote seminar on zink at last year's XDC, I brought up tiler performance as one of the deficiencies. Specifically this was in regard to how tilers need to maximize time spent inside renderpasses and avoid unnecessary load/store operations when beginning/ending those renderpasses, which required either some sort of Vulkan extension to enable deferred load/store op setting OR command stream parsing for GL.

While I did work on a number of Vulkan extensions this year, that wasn't one of them.

So it was that I implemented renderpass tracking for Threaded Context, which scans the GL command stream in the course of recording it for threaded dispatch. The CPU overhead is negligible (~5% on a couple `drawoverhead` cases and nothing noticeable in apps), while the performance gains are staggering (~10-15x speedup in AAA games). All in all, it was a painful process but one that has yielded great results.

The gist of it, as I've described in previous posts that I'm too lazy to find links for, is that framebuffer attachment access is accumulated during TC command recording such that zink is able to determine which load/store ops are needed. This works great so long as nothing unexpected splits the renderpass. "Unexpected" in this context refers to one of the following scenarios:
* zink receives a (transfer) command sequence which is impossible to reorder and must split the renderpass to execute copies/blits
* the app randomly flushes during rendering
* the GL frontend hits a TC synchronization point and halts the recording thread to wait for the driver thread to finish execution

The final issue remaining for renderpass tracking has been this third scenario: any time the GL frontend needs to sync TC, renderpass metadata is split. The splitting is such that a single renderpass becomes two because the driver must complete execution on the currently-recorded metadata in order to avoid deadlocking itself against the waiting GL frontend, but then the renderpass will continue after the sync. This happens in a very small number of scenarios, but one of them is extremely common.

Texture uploading.

# Texture Uploads: How Do They Work?
There are (currently) three methods by which TC can perform texture uploads:
* for small uploads, the data is enqueued and passed asynchronously to the driver thread
* for larger uploads:
  - if renderpass tracking is enabled and a renderpass is active, the upload will be sequenced into N strided uploads and passed asynchronously to the driver thread to avoid splitting renderpasses
  - otherwise TC syncs the driver thread and performs the upload directly

Eagle-eyed readers will notice that I've already handled the "problem" case described above; in order to avoid splitting renderpasses, I've written some handling which rewrites texture uploads into a sequence of N asynchronous buffer2image copies, where N is either 1 or `$height` depending on whether the source data's stride matches the image's stride. In the case where N is not 1, this can result in e.g., 4096 copy operations being enqueued for a 4096x4096 texture atlas. Even in the case where N is 1, it still adds an extra full copy of the texture data. While this is still more optimal than splitting a renderpass, it's not *optimal* in the absolute sense.

