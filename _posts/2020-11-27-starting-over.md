---
published: false
---
## New Game+

Up until now, I've been relying solely on my Intel laptop's onboard GPU for testing, and that's been great; Intel's drivers are robust as hell and have very few issues. On top of that, the rare occasions when I've found issues have led to a swift resolution.

Certainly I can't complain at all about my experience with Intel's hardware or software.

But now things are different and strange because I received in the mail a couple weeks ago a shiny AMD Radeon RX 5700XT.

Mostly in that it's a new codebase with new debugging tools and such.

Unlike when I started my zink journey earlier this year, however, I'm much better equipped to dive in and Get Things Done.

## Progress
This week's been a bit hectic as I worked to build my new machines, do holiday stuff, and then also get back into the project here. Nevertheless, significant progress has been made in a couple areas:
* I tested out and then dumped some very heavy-handed descriptor locks that had no impact on performance because they were never doing anything
* I fixed a major issue with barrier usage

The latter of these is what I'm going to talk about today, and I'm going to zoom in on one very specific handler since it's been a while since this blog has shown any actual code.

## Barrier Performance
When using Vulkan barriers, it's important to simultaneously:
* use enough of them that the underlying driver has enough information to transition resources into the states desired
* avoid using so many that your performance is garbage

Many months ago, I wrote a patch which aimed to address the second point while also not neglecting the first.

I succeeded in one of these goals.

The reason I didn't notice that I also failed in one of these goals until now is that ANV actually has weak barrier support. By this I mean that while ANV's barriers work fine and serve the expected purpose of changing resource layouts when necessary, it doesn't actually do anything with the [srcStageMask or dstStageMask](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdPipelineBarrier.html) parameters. Also, Intel drivers conveniently don't change an image's layout for different uses (i.e., GENERAL is the same as SHADER_READ), so screwing up these layouts doesn't really matter.

Is this a problem?

No.

ANV is great at doing ANV stuff, and zink has thus far been great at punting GL down the pipe so that ANV can blast out some (correct) pixels.

Consider the following code:
```
bool
zink_resource_image_needs_barrier(struct zink_resource *res, VkImageLayout new_layout, VkAccessFlags flags, VkPipelineStageFlags pipeline)
{
   if (!pipeline)
       pipeline = pipeline_dst_stage(new_layout);
    if (!flags)
       flags = access_dst_flags(new_layout);
    return res->layout != new_layout || (res->access & flags) != flags || (res->access_stage & pipeline) != pipeline;
}
```
This is a function I wrote for the purpose of no-oping redundant barriers. The idea here is that a barrier is unnecessary if it's applying the same layout with the same access (or a subset of access) for the same stages (or a subset of stages). The access and stage flags can optionally be omitted and filled in with default values for ease of use too.

Basic state tracking.

This worked great on ANV.

The problem, however, comes when trying to run this on a driver that really gets deep into putting those access and stage flags to work in order to optimize the resource's access.

RADV is such a driver.

Consider the following barrier sequence:
```
* VK_IMAGE_LAYOUT_GENERAL, VK_ACCESS_READ | VK_ACCESS_WRITE, VK_PIPELINE_STAGE_TRANSFER_BIT
* VK_IMAGE_LAYOUT_GENERAL, VK_ACCESS_READ | VK_ACCESS_WRITE, VK_PIPELINE_STAGE_TRANSFER_BIT
```
Obviously it's not desirable to use GENERAL as a layout, but that's how zink works for a couple cases at the moment, so it's a case that must be covered adequately. Going by the above filtering function, the second barrier has the same layout, the same access flags, and the same stage flags, so it gets ignored.

Conceptually, barriers in Vulkan are used for the purpose of informing the driver of dependencies between operations for both internal image layout (i.e., compressing/decompressing images for various usages) and synchronization. This means that if image `A` is written to in operation `O1` and then read from in operation `O2`, the user can either stall after `O1` or use one of the various synchronization methods provided by Vulkan to ensure the desired result. Given that this is GPU <-> GPU synchronization, that means either a semaphore or a pipeline barrier.

The above scenario seems at first glance to be a redundant barrier based on the state-tracking flags, but conceptually it isn't since it expresses a dependency between two operations which happen to use matching access.

## Refinement
After crashing my system a few times trying to do full piglit runs (seriously, don't ever try this with zink+radv at present if you're on similar hardware to mine), I came back to the barrier issue and started to rejigger this filter a bit.

The improved version is here:
```
bool
zink_resource_image_needs_barrier(struct zink_resource *res, VkImageLayout new_layout, VkAccessFlags flags, VkPipelineStageFlags pipeline)
{
   if (!pipeline)
      pipeline = pipeline_dst_stage(new_layout);
   if (!flags)
      flags = access_dst_flags(new_layout);
   return res->layout != new_layout || (res->access_stage & pipeline) != pipeline ||
          (res->access & flags) != flags ||
          (zink_resource_access_is_write(flags) && util_bitcount(flags) > 1);
}
```
This adds an extra check for a sequence of barriers where the new barrier has at least two access flags and one of them is write access. In this sense, the barrier dependency is ignored if the resource is doing `READ -> READ`, but `READ|WRITE -> READ|WRITE` will still be emitted like it should.

This change fixes a ton of unit tests, though I don't actually know how many since there's still some overall instability in various tests which cause my GPU to hang.

## RADVCAL
Certainly worth mentioning is that I've been working closely with the RADV developer community over the past few days, and they've been extremely helpful in both getting me started with debugging the driver and assisting with resolving some issues. In particular, keep your eyes on [these](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7751) [MRs](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7817) which also fix zink issues.

Stay tuned as always for more updates on all things Mike and zink.