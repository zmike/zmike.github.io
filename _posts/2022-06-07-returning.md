---
published: false
---
## I Remembered

After my last blog post I was so exhausted I had to take a week off, but I'm back. In the course of my blog-free week, I remembered the secret to blogging: blog before I start real work for the day.

It seems obvious, but once that code starts flowing, the prose ain't coming.

## Updates
It's been a busy couple weeks.

As I mentioned, I've been working on getting zink running on an Adreno chromebook using turnip—the open source Qualcomm driver. When I first fired up a CTS run, I was at well over 5000 failures. Today, two weeks later, that number is closer to 150. Performance in SuperTuxKart, the only app I have that's supposed to run well, is a bit lacking, but it's unclear whether this is anything related to zink. Details to come in future posts.

Some time ago I [implemented dmabuf support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16224) for lavapipe. This is finally landing, assuming CI doesn't get lost at the pub on the way to ramming the patches into the repo. Enjoy running Vulkan-based display servers in style.

Also in lavapipe-land, 1.3 conformance submissions are pending. While 1.2 conformance [went through](https://www.khronos.org/conformance/adopters/conformant-products/vulkan#submission_684) and was [blogged about](https://airlied.blogspot.com/2022/05/lavapipe-vulkan-12-conformant.html) to great acclaim, this unfortunately can't be listed in main or a current release since the driver has 1.2 conformance but advertises 1.3 support. The goalpost is moving, but we've got our hustle shoes on.

## Real Work
On occasion I manage to get real work done. A couple weeks ago, in the course of working on turnip support, I discovered something terrible: Qualcomm has no 64bit shader support.

This is different from old-time ANV, where 64bit support isn't in the hardware but is still handled correctly in the backend and so everything works even if I clumsily fumble some `dmat3` types into the GPU. On Qualcomm, such tomfoolery is Not Appreciated and will either crash or just fail altogether.

So how well did 64bit -> 32bit conversions work in zink?

Not well.

Very not well.

## A Plethora Of Failures
Before I get into all the methods by which zink fails, let's talk about it at a conceptual level: what 64bit operations are needed here?

There are two types of 64bit shader operations, `Int64` and `Float64`. At the API level, these correspond to doing 64bit integer and 64bit float operations respectively. At the functional level, however, they're a bit different. Due to how zink handles shader i/o, `Int64` is what determines support for 64bit descriptor interfaces as well as cross-stage interfaces. `Float64` is solely for handling ALU operations involving 64bit floating point values.

With that in mind, let's take a look at all the ways this failed:
* 64bit UBO loads
* 64bit SSBO loads
* 64bit SSBO stores
* 64bit shared memory loads
* 64bit shared memory stores
* 64bit variable loads
* 64bit variable stores
* 64bit XFB stores

Basically everything.

Oops.

## Fixing: Part One
There's a lot of code involved in addressing all these issues, so rather than delving too deeply into it (you can see the MR [here](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16669)), I'll go over the issues at a pseudo code level.

Like a whiteboard section of an interview.

Except useful.

First, let's take a look at descriptor handling. This is:
* 64bit UBO loads
* 64bit SSBO loads
* 64bit SSBO stores
* 64bit shared memory loads
* 64bit shared memory stores

All of these are handled in two phases. Initially, the load/store operation is rewritten. This involves two steps:
* rewrite the offset of the operation in terms of dwords rather than bytes (zink loads `array<uint>` variables)
* rewrite 64bit operations to 2x32 if 64bit support is not present

As expected, at least one of these was broken in various ways.

Offset handling was, generally speaking, fine. I got something right.

What I didn't get right, however, was the 64bit rewrites. While certain operations were being converted, I had (somehow) failed to catch the case of missing `Int64` support as a place where such conversions were required, and so the rewrites weren't being done. With that fixed, I discovered even more issues:
* more 64bit operations were being added in the course of converting from 64bit to 32bit
* non-scalar loads/stores broke lots of things

The first issue was simple to fix: instead of doing manual bitshifts and casts, use NIR intrinsics like `nir_op_pack_64_2x32` to handle rewrites since these can be optimized out upon meeting a matching `nir_op_unpack_64_2x32` later on.

The second issue was a little trickier, but it mostly just amounted to forcing scalarization during the rewrite process to avoid issues.

Except there were other issues, because of course there were.

Some of the optimization passes required for all this to work properly weren't handling atomic load/store ops correctly and were deleting loads that occurred after atomic stores. This spawned a [massive amount of galaxy brain-level discussion](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16638) that culminated in some patches that might land someday.