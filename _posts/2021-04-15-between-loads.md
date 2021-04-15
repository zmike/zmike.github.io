---
published: false
---
## TFW Long Game Loads

I'm typing this up between loads and runs of various games I'm testing, since bug reports for games are somehow already a thing, and there's a lot of them.

The worst part about testing games is the unbelievably long load times (and startup videos) most of them have, not to mention those long, panning camera shots at the start of the game before gameplay begins and I can start crashing.

But this isn't a post about games.

No, no, there's plenty of time for such things.

This is a combo post: part roundup because blogging has been sporadic the past couple weeks, and part feature.

## The Roundup
The big Mesa 21.1 branchpoint happened yesterday, and I'm pretty pleased with the state of zink in this upcoming release.

**Things you should expect to see:**
* GL 4.6
* ES 3.1
* Reasonable performance in many cases

**Things you should not expect to see:**
* Most (any?) AAA games working; I've kept GL compat contexts clamped to 3.0 in a certainly-futile attempt to cut down on the absolute deluge of bug tickets I'm expecting once everyone tries to run their favorite games/emulators/whathaveyou with the shipped version
  * This functionality remains enabled in zink-wip and will be dumped into mainline soon
* ???
* Tough to say, honestly, since this is effectively a version of zink that is, to me, 5-6 months old with a few other patches sprinkled in here and there

And here's the zink-wip roundup from the past however long since I did the last one:
* I [doubled blending performance](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/10180) in many cases by fixing an incredibly old TODO item regarding using linear tiled images for scanout; this got rushed into the 21.1 release solely to avoid super embarrassing numbers on Phoronix benchmarks.

Yeah, I'm talking to you.

* I fixed a ton of bugs. Like, actually a ton. [Tomb Raider](https://store.steampowered.com/app/203160/Tomb_Raider/) went from an all-you-can-crash buffet to...well, I'll leave it as a fun surprise for anyone feeling especially intrepid, but it's definitely playable. So are things that use queries. Don't ask.

* There's totally some other stuff, but I'm too fried to remember it.

## The Real Post
Everything up there was just fluff, but this is where the post gets real.

Zink supports formats with alpha-to-one, e.g., RGBX, BGRX, and even (on zink-wip, very illegal) XRGB and XBGR. This is handy for 24bit visuals, such as (probably) all your windows in Xorg. But the method by which these formats are supported comes with its own issues, part of which I'd fixed some time ago in my quest to reduce GPU overhead, and the other part I discovered more recently.

In zink, an alpha-to-one format is just the equivalent format with alpha. So RGBX is just RGBA. This is due to Vulkan not having format equivalents for these types; when sampling from them, a swizzle is applied to force the alpha channel to the maximum value, which yields the correct result.

But what happens when an RGBX framebuffer attachment is used in a blending operation?

Let's look at `VK_BLEND_OP_ADD` as a very simple example. The spec defines this operation as:

`As0 × Sa + Ad × Da`

That's alpha-of-src times src-alpha-blend-factor plus alpha-of-dest times dest-alpha-blend-factor yielding the resulting pixel color that gets written to the attachment.

But what if the dest value is expected to always one, and the actual buffer is always zero because its alpha channel is never written?

Such is the case with RGBX and the like, and so more steps are required here for full emulation.

## The Real Roundup
Here's how I went about solving the issue.

First, framebuffer attachments have to be monitored when they're updated, and any time an alpha-to-one attachment is bound, I set a bitflag for it in the pipeline state. This then triggers a pipeline update for the blend state at the time of draw, where I apply clamping to the appropriate blend factors like so:

```c
if (state->zero_alpha_attachments) {
   for (unsigned i = 0; i < state->num_attachments; i++) {
      blend_att[i] = state->blend_state->attachments[i];
      if (state->zero_alpha_attachments & BITFIELD_BIT(i)) {
         blend_att[i].dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
         blend_att[i].srcColorBlendFactor = clamp_zero_blend_factor(blend_att[i].srcColorBlendFactor);
         blend_att[i].dstColorBlendFactor = clamp_zero_blend_factor(blend_att[i].dstColorBlendFactor);
      }
   }
   blend_state.pAttachments = blend_att;
} else
   blend_state.pAttachments = state->blend_state->attachments;
```

For any of the attachments in the bitfield, three clamps are performed:
* `dstAlphaBlendFactor` is clamped to zero, because there will never be any contribution from the dest component of alpha blending
* `srcColorBlendFactor` and `dstColorBlendFactor` are both clamped using the following check:

```c
if (f == VK_BLEND_FACTOR_ONE_MINUS_DST_ALPHA)
   return VK_BLEND_FACTOR_ZERO;
if (f == VK_BLEND_FACTOR_DST_ALPHA)
   return VK_BLEND_FACTOR_ONE;
return f;
```

Thus, alpha blending is a passthrough operation from the `src` component, and for color blending, the `dest` component is always one or zero. This yields correct results in piglit's `spec@arb_texture_float@fbo-blending-formats` test, and also potentially enables the hardware to employ some optimizations to reduce the burden of blending.

Exciting.