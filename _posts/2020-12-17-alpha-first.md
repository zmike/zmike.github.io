---
published: false
---
## The Name Of The Game

...is emulation.

With blit-based transfers working, I checked my RPCS3 flamegraph again to see the massive performance improvements that I'd no doubt be seeing:

[![fg.png]({{site.url}}/assets/bioshock/fg.png)]({{site.url}}/assets/bioshock/fg.png)

Except there were none.

Closer examination revealed that this was due to the app using ARGB formats for its PBOs. Referencing that against [VkFormat](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkFormat.html) led to a further problem: ARGB and ABGR are not explicitly supported by Vulkan.

This wasn't exactly going to be an easy fix, but it wouldn't prove too challenging either.

## Swizzle Me Timbers
Yes, swizzles. In layman's terms, a swizzle is mapping a given component to another component, for example in an RGBA-ordered format, using a WXYZ swizzle would result in a reordering of the 0-indexed components to 3012, or ARGB.

Gallium, when using blit-based transfers, provides a lot of opportunities to use swizzles, specifically by having a lot of blits go through a `u_blitter` codepath that translates blits into quad draws with a sampled image.

Thus, by applying an ARGB/ABGR emulation swizzle to each of these codepaths, I can drop in native-ish support under the hood of the driver by internally reporting ARGB as RGBA and ABGR as BGRA.

In pseudocode, the ARGB path looks something like this:
```c
 unsigned char dst_swiz[4];

 if (src_is_argb) {
    unsigned char reverse_alpha[] = {
       PIPE_SWIZZLE_Y,
       PIPE_SWIZZLE_Z,
       PIPE_SWIZZLE_W,
       PIPE_SWIZZLE_X,
    };
    /* compose swizzle with alpha at the end */
    util_format_compose_swizzles(original_swizzle, reverse_alpha, dst_swiz);
 } else if (dst_is_argb) {
    unsigned char reverse_alpha[] = {
       PIPE_SWIZZLE_W,
       PIPE_SWIZZLE_X,
       PIPE_SWIZZLE_Y,
       PIPE_SWIZZLE_Z,
    };
    /* compose swizzle with alpha at the start */
    util_format_compose_swizzles(original_swizzle, reverse_alpha, dst_swiz);
}
```
The original swizzle is composed with the alpha-reversing swizzle to generate a swizzle that translates the resource's internal ARGB data into RGBA data (or vice versa) like the Vulkan driver is expecting it to be.

From there, the only restriction is that this emulation is prohibited in texel buffers due to there not being a direct method of applying a swizzle to that codepath. Sure, I could do the swizzle in the shader as a variant, but then this leads to more shader variants and more pipeline objects, so it's simpler to just claim no support here and let gallium figure things out using other codepaths.

## Progress?
Would this be enough to finally get some frames moving?

Find out tomorrow in the conclusion to this SGC miniseries.