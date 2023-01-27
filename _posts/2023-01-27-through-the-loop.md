---
published: false
---
## A New Level Of Speed

I know everyone's been eagerly awaiting the return of the pasta maker.

The wait is over.

But today we're going to move away from those dangerous, addictivive synthetic benchmarks to look at a different kind of speed. That's right. Today we're looking at **pipeline compile speed**. Some of you are scoffing, mouse pointer already inching towards the close button on the tab.

Pipeline compile speed in the current year? Why should anyone care when we have great tools like [Fossilize](https://github.com/ValveSoftware/Fossilize/) that can precompile everything for a game over the course of several hours to ensure games have no stuttering?

It turns out there's at least one type of pipeline compile that still matters going forward. Specifically, I'm talking about fast-linked pipelines using [VK_EXT_graphics_pipeline_library](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_graphics_pipeline_library.html).

Let's get an appetizer going, some exposition under our belts before we get to the spaghetti we're all craving.

## Pipelines: They're In Your Games
All my readers are graphics experts. It won't come as any surprise when I say that a **pipeline** is a program containing shaders which is used by the GPU. And you all know how `VK_EXT_graphics_pipeline_library` enables compiling partial pipelines into libraries that can then be combined into a full pipeline. None of you need a refresher on this, and we all acknowledge that I'm just padding out the word count of this post for posterity.

Some of you experts, however, have been so deep into getting those green triangles on the screen to pass various unit tests that you might not be fully aware of the fast-linking property of `VK_EXT_graphics_pipeline_library`.

In general, everyone knows that compiling shaders during gameplay is (usually) bad. This is (usually) what causes stuttering: the compilation of a pipeline takes longer than the available time to draw the frame, and rendering blocks until it completes. The fast-linking property of `VK_EXT_graphics_pipeline_library` changes this paradigm by enabling pipelines, e.g., for shader variants, to be created fast enough to avoid stuttering.

Typically, this is utilized in applications through the following process:
* create pipeline libraries from shaders during game startup or load screen
* wait until pipeline is needed at draw-time
* fast-link final pipeline and use for draw
* background compile an optimized pipeline for future use

In this way, no draw is blocked by a pipeline creation, and optimized pipelines are still used for the majority of GPU operations.

## But Why...
...would I care about this if I have Fossilize and a brand new gaming supercomputer with 256 cores running at 12GHz?

I know you're wondering, and the answer is simple: not everyone has these things.

Some people don't have extremely modern computers, which means Fossilize pre-compile of shaders can take hours. Who wants to sit around waiting that long to play a game they just downloaded?

Some games don't use Fossilize, which means there's no pre-compile. In these situations, there are two options:
* compile *all* pipelines during load screens
* compile on-demand during rendering

The former option here gives us load times that remind us of the original Skyrim release. The latter probably yields stuttering.

Thus, `VK_EXT_graphics_pipeline_library` (henceforth GPL) with fast-linking.

## The Obvious Problem
What does the "fast" in fast-linking really mean?

How fast is "fast"?

These are great questions that nobody knows the answer to. The only limitation here is that "fast" has to be "fast enough" to avoid stuttering.

Given that RADV is in the process of bringing up GPL for general use, and given that Zink is relying on fast-linking to eliminate compile stuttering, I thought I'd take out my perf magnifying glass and see what I found.

[![eyeofsauron.jpg]({{site.url}}/assets/eyeofsauron.jpg)]({{site.url}}/assets/eyeofsauron.jpg)

## Uh-oh
Obviously we wouldn't be advertising fast-linking on RADV if it wasn't fast.

Obviously.

It goes without saying that we care about performance. No credible driver developer would advertise a performance-related feature if it wasn't performant.

**RIGHT?**

And it's not like I tried running Tomb Raider on zink and discovered that the so-called "fast"-link pipelines were being created at a non-fast speed. That would be insane to even consider—I mean, it's literally in the name of the feature, so if using it caused the game to stutter, or if, for example, I was seeing "fast"-link pipelines being created in **10ms+**...

[![mesa.png]({{site.url}}/assets/mesa.png)]({{site.url}}/assets/mesa.png)

Surely I didn't see that though.

[![rotated_mesa.png]({{site.url}}/assets/rotated_mesa.png)]({{site.url}}/assets/rotated_mesa.png)

Surely I didn't see **fast**-link pipelines taking more than an entire frame's worth of time to create.

## It's Fine
Long-time readers know that this is fine. I'm unperturbed by seeing numbers like this, and I can just file a ticket and move on with my life like a normal per—

[![angry_mesa.png]({{site.url}}/assets/angry_mesa.png)]({{site.url}}/assets/angry_mesa.png)

OBVIOUSLY I CAN'T.

Obviously.

And just as obviously I had to get a second opinion on this, which is why I took my testing over to the only game I know which uses GPL with fast-link: ~~[3D Pinball: Space Cadet](https://apps.microsoft.com/store/detail/3d-pinball---space-cadet/9MT92DXZJ11R)~~ DOTA 2!

Naturally it would be DOTA2, along with other Source Engine 2 games, that uses this functionality.

Thus, I fired up my game, and faster than I could scream **MID OR MEEPO** into my mic, I saw the unthinkable spewing out in my console:

```
COMPILE 11425115
COMPILE 39491
COMPILE 11716326
COMPILE 35963
COMPILE 11057200
COMPILE 37115
COMPILE 10738436
```

Yes, those are all "fast"-linked pipeline compile times in nanoseconds.

Yes, half of those are taking more than 10ms.

[![pastamaker.jpg]({{site.url}}/assets/pastamaker.jpg)]({{site.url}}/assets/pastamaker.jpg)

## First Steps
The first step is always admitting that you have a problem, but I don't have a problem. I'm fine. Not upset at all. Don't read more into it.

As mentioned above, we have great tools in the Vulkan ecosystem like [Fossilize](https://github.com/ValveSoftware/Fossilize/) to capture pipelines and replay them outside of applications. This was going to be a great help.

I thought.

I fired up a 32bit build of Fossilize, set it to run on Tomb Raider, and immediately it exploded.

[![fastlink-works.png]({{site.url}}/assets/fastlink/fastlink-works.png)]({{site.url}}/assets/fastlink/fastlink-works.png)


