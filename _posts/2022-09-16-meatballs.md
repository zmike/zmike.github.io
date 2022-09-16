---
published: false
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

## 