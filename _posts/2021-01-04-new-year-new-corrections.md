---
published: true
---
## It Happens

As long-time readers of the blog know, SGC is a safe space where making mistakes is not only accepted, it's a way of life. So it is once again that I need to amend statements previously made regarding Xorg synchronization after Michel Dänzer, also known for anchoring the award-winning series Why Is My MR Failing CI Today?, pointed out that while I was indeed addressing the correct problem, I was addressing it from the wrong side.

## Looking Closer
The issue here is that WSI synchronizes with the display server using a file descriptor for the swapchain image that the Vulkan driver manages. But what if the Vulkan driver never configures itself to be used for WSI (genuine or faked) in the first place?

Yes, this indeed appeared to be the true problem. Iago Toral Quiroga [added handling for this](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7378) specific to the V3DV driver back in October, and it's the same mechanism: setting up a Mesa-internal struct during resource initialization.

So I extended this to the ANV codepath and...

And obviously it didn't work.

But why was this the case?

A script-based `git blame` revealed that ANV has a different handling for implicit sync than other Vulkan drivers. After a [well-hidden](https://gitlab.freedesktop.org/mesa/mesa/-/commit/ccb7d606f1a2939d5a784f1ec491cffc62e8f814) patch, ANV relies entirely on a struct attached to `VkSubmitInfo` which contains the swapchain image's memory pointer in order to handle implicit sync. Thus by attaching a `wsi_memory_signal_submit_info` struct, everything was resolved.

Is it a great fix? No. Does it work? Yes.

## Questions
**If ANV wasn't configuring itself to handle implicit sync, why was poll() working?**

Luck.

**Why does RADV work without any of this?**

Also probably luck.
