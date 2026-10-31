---
layout: post
title: Friday NV Fight
published: true
---

# A Final Frontier
I've looked at perf in a lot of places. I looked between the seats, under the couch, in the jar where I usually leave it, but until recently I'd never looked in one place. It was too scary. Too complex.

Seriously, how do you even build mesa on Windows?

I don't know, and I don't feel my life will be enriched by finding out. But I was still curious about one thing: How does zink stack up against the industry-standard, the titan of GL, the Windows NVIDIA proprietary driver?

## In The Green Corner
The reigning heavyweight champ, the driver preferred by gamers and industry afficianados alike! They wrote (many of) the specs, they implemented (many of) the extensions, their driver has no bugs at all (because apps have collectively turned these bugs into a standard-but-wrong dialect of OpenGL)!

NVIDIA, v617.14!

## In The Open Corner

The only open source graphics stack out there using a quarterly versioning scheme! Despite shipping a new major release only a month ago, that release is already obsolete! These drivers are hungry for perf, and they'll stop at nothing to get it!

Mesa, \<my own wip set of patches which are mostly upstream but not all upstream and so I do not have a specific commit hash\>!

# Epic Matchup

[![nv-vs-zink.png]({{site.url}}/assets/nv-vs-zink.png)]({{site.url}}/assets/nv-vs-zink.png)

I won't tell you which one I am in the picture, but my office does have a helmets-required policy. Setup went fine (computer, set up the Windows build!) while I did real work, and then it was off to the races.

The first results were... Well, they weren't as bad as expected. But there were some inconsistencies, which is to say that zink wasn't winning hard enough. Or at all in a couple cases. It came down to the wire with the weekend looming, but ultimately I was able to learn the underhanded tricks that NVIDIA GL was using to stay ahead.

# Questionably Ethical Optimizations

There are (apparently?) some real weaknesses in NVIDIA hardware, specifically with line drawing. To get around this, they cut some corners.

First up: smooth lines on color attachments with float formats. Apparently this is 50x slower (literally) than using rectangular lines. So they just draw rectangular lines. I know what you're thinking—You're thinking Mike, that's gotta be a spec violation. And yes, it is. And I know what you're thinking next—You're thinking Mike, how can they possibly pass CTS like that? And the answer is easily, because there are (very conveniently) zero tests anywhere in the ecosystem for this combination. Not in CTS, not in piglit, not even under the couch.

DJ Khaled (anotha one): list topology draws with primitive restart enabled are about 5x slower because processing the restart indices is somehow very, very slow. NVIDIA gets around this by preprocessing index buffers in compute shaders any time this combination occurs. Nothing illegal about it since the solution is conformant, it's just a little weird that they have to.

# Monkey See, Monkey Do

Yeah, I did the same things. Even on a pure Mesa stack, NVK is still 3-9x slower for list topology restarts and 5x slower for the float-format lines, so this is likely something at the hardware level that won't be fixed by a VK driver. The float-format change was trivial, but the index buffer rewrite ended up being more complex. I got about halfway through trying to convert `util_prim_restart_convert_to_direct` to a compute shader before I started wondering whether this was the optimal choice.

Turns out it wasn't. A more optimal choice is a three-pass shader using subgroups which consume chunks of the input index buffer:
* count the primitives that will actually be drawn and set `flag` if there are any which contain both real and restart indices
* prefix-sum the counts from the previous pass into the offsets used by the next pass, then write the eventual draw command (total primitives drawn, or the original draw if previous pass set `flag`)
* write the indices using the offsets from the second pass (or copy all indices without change if the `flag` is set)

Then use the resulting buffer for an indirect draw.

# Power Overwhelming

## SPECviewperf 2020

| viewset | tests | native (best) | zink | zink / native |
|---|---:|---:|---:|---:|
| catia | 8 | 124.83 | 228.94 | 1.83 |
| creo | 13 | 121.04 | 191.64 | 1.58 |
| energy | 6 | 91.93 | 111.71 | 1.22 |
| maya | 2 | 693.69 | 738.16 | 1.06 |
| medical | 10 | 65.39 | 77.50 | 1.19 |
| snx | 10 | 40.13 | 865.09 | 21.56 |
| sw | 10 | 457.11 | 548.98 | 1.20 |

**Geomean:** 2.194 over 59 tests.

## SPECviewperf 15

| viewset | tests | native (best) | zink | zink / native |
|---|---:|---:|---:|---:|
| blender-01 | 7 | 99.64 | 126.31 | 1.27 |
| catia-07 | 5 | 80.68 | 106.16 | 1.32 |
| creo-04 | 13 | 118.60 | 199.99 | 1.69 |
| energy-04 | 6 | 91.64 | 110.16 | 1.20 |
| maya-07 | 7 | 234.01 | 251.66 | 1.08 |
| medical-04 | 10 | 64.78 | 78.71 | 1.22 |
| snx-05 | 8 | 45.29 | 68.90 | 1.52 |
| solidworks-08 | 8 | 55.28 | 67.52 | 1.22 |

**Geomean:** 1.332 over 64 tests.

## Superposition

| preset | native | zink | zink / native |
|---|---:|---:|---:|
| Medium | 28928 | 34668 | 1.20 |
| Extreme | 14837 | 14770 | 1.00 |

The results are pretty one-sided. I won't claim that they hold for every possible case, but viewperf and superposition combined give a reasonable picture of OpenGL performance. If you want the best GL experience on your green hardware and don't need NV-specific extensions, you should try using zink.

# ...With Some Caveats

The cases here were run three ways:
* NVIDIA (threaded optimizations disabled)
* NVIDIA (threaded optimizations enabled)
* zink (`ZINK_DESCRIPTORS=lazy`)

I used the best of the NVIDIA results (though they didn't vary hugely between the modes), but I didn't use `ZINK_DESCRIPTORS=db` because I was pressed for time and this is known (supposedly) to be slightly slower on NVIDIA hardware. Probably not more than 5% from my testing, so it only affects the margins. I'll probably just build out ARB_separate_shader_object handling for 1.0 descriptors (currently only supported for db because I was lazy) and then switch NVIDIA over to it by default. I imagine I could bang that out in an afternoon and a hundred lines of code. No, I still don't plan to add descriptor heap support; I don't need to maintain another 2k+ lines of descriptor code when the current solution works fine.

There are also some patches which probably won't get upstreamed until next year. In particular, the ray-marching loops in viewperf's energy viewsets perform extremely poorly due to loop optimization deficiencies in NVIDIA's compiler stack. I have some hacky stuff to work around this, but I'd rather wait for some upcoming NIR work that's being done by people more compiler-y. It only seems to affect these very specific shaders anyway, so unless you're doing that type of work you won't notice. But if you are, it's about 40% worse until Mesa lands more loop optimizations. NVK is unaffected.

Anyway, mesa's pretty fast. Oh yeah, and [here's WGL glthread support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/45387).
