---
published: false
---
## Another Milestone

A while ago, I blogged about how zink was leveraging `VK_EXT_graphics_pipeline_library` to avoid mid-frame shader compilation, AKA stuttering. This was all good and accurate, in that the code existed, it worked, and when the right paths were taken, there was no mid-frame shader compiling.

The problem, of course, is that these paths were never taken.