---
published: false
---
## A New Level Of Speed

I know everyone's been eagerly awaiting the return of the pasta maker.

The wait is over.

But today we're going to move away from those dangerous, addictivive synthetic benchmarks to look at a different kind of speed. That's right. Today we're looking at **pipeline compile speed**. Some of you are scoffing, mouse pointer already inching towards the close button on the tab.

Pipeline compile speed in the current year? Why should anyone care when we have great tools like [Fossilize](https://github.com/ValveSoftware/Fossilize/) that can precompile everything for a game over the course of several hours to ensure games have no stuttering?

It turns out there's at least one type of pipeline compile that still matters going forward. Specifically, I'm talking about fast-linked pipelines using [VK_EXT_graphics_pipeline_library](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_graphics_pipeline_library.html).

Let's get some exposition under our belts before we get to the spaghetti.