---
published: false
---
## Another Milestone

A while ago, I blogged about how zink was leveraging `VK_EXT_graphics_pipeline_library` to avoid mid-frame shader compilation, AKA stuttering. This was all good and accurate, in that the code existed, it worked, and when the right paths were taken, there was no mid-frame shader compiling.

The problem, of course, is that these paths were never taken. Who could have guessed: there were bugs.

These bugs are now fixed, however, and so there should be **no more stuttering ever with zink**.

Period.

## Except
There's this little extension I hate called [ARB_separate_shader_objects](https://registry.khronos.org/OpenGL/extensions/ARB/ARB_separate_shader_objects.txt). It allows shaders to be created *separately* and then linked together at runtime in a performant manner that doesn't stutter.

If you're thinking this sounds a lot like GPL fast-linking, you're not wrong.

If, however, you've noticed the obvious flaw in this thinking, you're also not wrong.

## 