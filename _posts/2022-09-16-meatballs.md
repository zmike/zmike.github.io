---
published: true
---
## Before We Begin

We all know where this post is going to end up.

We've been there before.

We'll be there again.

But we're not there yet. And before we get there, I need everyone to be absolutely serious for a little while. No memes, no jokes, no puns, just straight talk from me to you.

Let's talk about Intel.

I know what you're thinking.

If you work for Intel, you're thinking _Oh no_.

If you don't, you're thinking _Oh no_. Or possibly getting ready to post some comments about ANV not supporting features, or having bugs, or any of the common refrains that I've seen so often around the internet.

The fact remains, however, that ANV is the reference zink hardware driver. It has been ever since I started working on zink. The reason for this is simple: it's what I have on the machine I develop with, and when I started working on zink, it was the driver that was furthest along.

It's therefore no exaggeration to say that without everything ANV and the Intel Mesa team brings to the table, zink wouldn't be nearly as far along as it is.

And neither would RADV.

For those of you unaware, at the top of many RADV code files is this comment:

```c
 * based in part on anv driver which is:
 * Copyright © 2015 Intel Corporation
```

That's right. RADV originally started out reusing a lot of code written by Intel. It's no exaggeration to say that RADV wouldn't be nearly as far along without Intel's Mesa team.

Let's go deeper though. Using LWN's Git Data Miner ([gitdm](git://git.lwn.net/gitdm.git)) on `mesa/src/compiler` (which includes GLSL, SPIR-V, and NIR), let's see who the top contributors are to the heart of Mesa:

|Name|Commit Count|Percentage of Total|Affiliation|
|--------|----|------|----|
|Jason Ekstrand|1429|19.7%|Intel/Collabora|
|Timothy Arceri|714|9.9%|Collabora/Valve|
|Ian Romanick|577|8.0%|Intel|
|Rhys Perry|298|4.1%|Valve|
|Caio Oliveira|270|3.7%|Intel|
|Emma Anholt|268|3.7%|Google|
|Marek Olšák|260|3.6%|AMD|
|Kenneth Graunke|224|3.1%|Intel|
|Samuel Pitoiset|176|2.4%|Valve
|Connor Abbott|168|2.3%|Intel/Valve|

Again we see that Intel's Mesa team has been hard at work over the years to enable the things that we all now take for granted.

So now you know why every time I see someone saying that ANV is bad, or Intel engineers aren't doing everything they can, or the constant memeposting about any number of related issues, it hurts. I've been relying on the fruits of the Intel Mesa team's labor for years now, and so have you.

If anything, we should be cheering them on to continue doing the great work they've been doing all along.

Thanks, Intel Mesa team.

I appreciate you.

And so should everyone else.

## But Now
It's time for another round of

where

[![mesa.png]({{site.url}}/assets/mesa.png)]({{site.url}}/assets/mesa.png)

is

[![rotated_mesa.png]({{site.url}}/assets/rotated_mesa.png)]({{site.url}}/assets/rotated_mesa.png)

**MY SPAGHETTI?**

[![angry_mesa.png]({{site.url}}/assets/angry_mesa.png)]({{site.url}}/assets/angry_mesa.png)

...and meatballs, because obviously, as an American, I'm obligated to add condiments to everything. Mainly melted cheese.

In today's edition of WIMS, I'll be delving into the depths of Intel's Mesa Vulkan driver to see what kinds of spaghetti they're growing and how much of it I can eat before they catch me.

The first step, as always, is to pull out my trusty `perf` and...

Once again, I know what you're thinking.

[![1.png]({{site.url}}/assets/chopper/1.png)]({{site.url}}/assets/chopper/1.png)

You're thinking that I can't just use `perf` and *profile* my way out real performance issues.

But what if I showed you this from my Intel Icelake machine:

```
$ ./vkoverhead -test 0 -duration 3 -output-only
44854
```

Does that seem like a good number? Something reasonable for the ancestor of all Mesa Vulkan drivers? Let's check out a flamegraph, just for the sake of this is my blog and I'm gonna do it whether you want me to or not.

[![perf.png]({{site.url}}/assets/meatballs/perf.png)]({{site.url}}/assets/meatballs/perf.png)

And now I'm going to tell you

[![2.png]({{site.url}}/assets/chopper/2.png)]({{site.url}}/assets/chopper/2.png)

that I can too use `perf` for everything, and profiling is both a legitimate and useful way to improve any and all kinds of performance.

But looking at that graph, there's something obvious that stands out here. Why is [gfx11_emit_apply_pipe_flushes()](https://gitlab.freedesktop.org/mesa/mesa/-/blob/d5dedecfe7ee90cf220da75b5ac21d9f651294bf/src/intel/vulkan/genX_cmd_buffer.c#L1806) showing up twice?

[![perfbad.png]({{site.url}}/assets/meatballs/perfbad.png)]({{site.url}}/assets/meatballs/perfbad.png)

The answer may surprise you.

Let's go back in time. I want everyone to pretend that it's early 2021 again. Photos are still being taken in black and white by hipsters, nobody has toilet paper, and everything is generally worse. Vulkan is also worse because I haven't shipped any of my extensions. Extensions like [VK_EXT_multi_draw](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_multi_draw.html), which I implemented for a number of Mesa drivers. Drivers like RADV. And Lavapipe.

And ANV.

You might see where I'm going with this.

Yes, it was all the way back in [June of 2021](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11531) that I implemented this extension for ANV. And at the time, everything was worse, including the world's lack of our lord and savior, our premier optimizing tool, that project I'm going to keep plugging in my blog until the end of time, [vkoverhead](https://github.com/zmike/vkoverhead). Back then, the only way anyone could know how their driver performed at the CPU level was to use `drawoverhead` through zink.

And zink wasn't as fast then as it is now; whoever was working on it back then was a total fucking idiot who couldn't optimize his way out of a `for` loop.

So it's not going to surprise anyone, and it's not like anyone would even be mad if they found out that in the course of those trivial, barely even noticeable changes to the driver, I also maybe sorta kinda [increased ANV's draw command recording overhead](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11531/diffs?commit_id=1e39f2c1994abb7183ebc05de29e11561c388eb5). But it's not like it was by any big amount or anything like that. I mean, it's not like _I_ was a total fucking idiot back... back then... Well, it's not like I wouldn't have realized it even if I did accidentally nerf the driver that I relied upon for my daily use by an amount that was in any way significant, right?

It's not like it was a full [30%](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18637/diffs?commit_id=38f0ed795ab526be20ace9f90dac938340f2b758) or anything like that.

I, uh... I have to go for now, but it's not like Intel's Mesa team just rolled up on my house or anything like that because taht would be kinda crazy haha okay but brb

## Barely Hanging On
**S**o I'm back and I'm totally fin**e**, do**n**'t ask if I nee**d** a gofundme for my medical bills or anyt**h**ing b**e**cause there definite**l**y aren't any and I'm just great with**p** both hands still firmly attached and at least eight fingers in total between the two, which is, if you think about it, really just way more than anyone actually needs to write software.

And you know, I was totally gonna stop now with these new results

```
$ ./vkoverhead -test 0 -duration 3 -output-only
59973
```

but haha you know I'm being told that I'd better keep going **or else** hah so we're just gonna uh gonna keep getting in there and seeing what we can find, if that's all right with everyone? Yeah? Cool? Great, so, uh, yeah, let's just check out another flamegraph real quick

[![perf2.png]({{site.url}}/assets/meatballs/perf2.png)]({{site.url}}/assets/meatballs/perf2.png)

And that's looking just great. Yeah, it's really great, isn't it? It's not? Okay, well, we've got some options. Uh, yeah, lots of options. Not like **I need help** figuring out what they are or anything, since this is what I'm good at. But there's this big old [gfx11_cmd_buffer_flush_state](https://gitlab.freedesktop.org/mesa/mesa/-/blob/d5dedecfe7ee90cf220da75b5ac21d9f651294bf/src/intel/vulkan/genX_cmd_buffer.c#L3394) call sticking out like a sore thumb—and I've obviously still got two of those like everyone else—so let's see what we can do about that.

Yup, well, it's looking like [inlining those functions](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18637/diffs?commit_id=299dad5776c07898e26a7c10b3fafdbea2a2cd1a) is...

What?

[![3.png]({{site.url}}/assets/chopper/3.png)]({{site.url}}/assets/chopper/3.png)

You're telling me I can't just inline everything?

[![4.png]({{site.url}}/assets/chopper/4.png)]({{site.url}}/assets/chopper/4.png)

But it's improving performance!

```
$ ./vkoverhead -test 0 -duration 3 -output-only
71192
```

And that's great, isn't it? An extra **60%+ draw throughput on ANV**? Yeah? Oh, okay, so we're done? Yeah, no, that's great, that's really great. I'm glad we could come to an understanding. No, no, it was no trouble. No, I don't think we'll need to do this again, either. Learned a lot today. Just learning all around.

## Results
Well, we made it through that—not like there was anything special or unusual happening today, just an ordinary turn of phrase—and everything's fine.

[![5.png]({{site.url}}/assets/chopper/5.png)]({{site.url}}/assets/chopper/5.png)

But what does this mean for real-world performance results, you're asking?

[![6.png]({{site.url}}/assets/chopper/6.png)]({{site.url}}/assets/chopper/6.png)

I'm just micro-optimizing for the benchmarks that I write, not saying **this will double your Cyberpunk 2077 FPS**.

I'm not saying it.
