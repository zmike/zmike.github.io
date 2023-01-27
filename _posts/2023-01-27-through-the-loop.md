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