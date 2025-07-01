# Timelines

It's hot out. I know this because Big Triangle allowed me a peek through my three-sided window for good behavior, and all the pixels were red. Sure am glad I'm inside.

Today's a new day in a new month, which means it's time to talk about new GL stuff. I'm allowed to do that once in a while, even though GL stuff is never actually new. In this post we're going to be looking at [GL_NV_timeline_semaphore](https://registry.khronos.org/OpenGL/extensions/NV/NV_timeline_semaphore.txt), an extension everyone has definitely heard of.

Mesa has supported [GL_EXT_external_objects](https://registry.khronos.org/OpenGL/extensions/EXT/EXT_external_objects.txt) for a long while, and it's no exaggeration to say that this is the reference implementation: there are no proprietary drivers of which I'm aware that can pass the super-strict piglit tests we've accumulated over the years. Yes, that includes Green Triangle. Also Red Triangle, but we knew that already--it's in the name.

[This MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/35866) adds support for importing and using Vulkan timeline semaphores into GL, which further improves interop-reliant workflows by eliminating binary semaphore requirements. Zink supports it anywhere that additionally supports [VK_KHR_timeline_semaphore](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_timeline_semaphore.html), which is to say that any platform capable of supporting the base external objects spec will also support this.

For testing, we get to have even more fun with the industry-standard [ping-pong test](https://gitlab.freedesktop.org/mesa/piglit/-/merge_requests/1022) originally contributed by @gfxstrand. This verifies that timeline operations function as expected on every side of the API divide.

Next up: more optimizations. How fast is too fast?