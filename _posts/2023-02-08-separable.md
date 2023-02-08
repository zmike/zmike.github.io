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

## Separate Shaders In Vulkan
This is not a thing, but it needs to be a thing. Specifically it needs to be a thing because our favorite triangle princess, Tomb Raider (2013), uses SSO extensively, which is [why the fps is bad](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8223).

Let's think about how this would work. GPL allows for splitting up the pipeline into four sections:
* vertex/input assembly
* non-fragment shaders
* fragment shader
* multisample/blend

Zink already uses this for normal shader programs. It creates partial pipeline libraries that look like this:
* vertex/input assembly
* all shaders
* multisample/blend

The `all shaders` library is generated asynchronously during load screens, the other two are generated on-demand, and the whole thing gets fast-linked together so quickly that there is (now) zero stuttering.

Logically, supporting SSO from here should just mean expanding `all shaders` to a pair of pipeline libraries that can be created asynchronously:
* vertex shader
* fragment shader

This pair of shader libraries can then be fast-linked into the `all shaders` intermediate library, and the usual codepaths can then be taken.