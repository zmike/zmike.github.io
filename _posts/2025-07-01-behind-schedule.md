# Timelines

It's hot out. I know this because Big Triangle allowed me a peek through my three-sided window for good behavior, and all the pixels were red. Sure am glad I'm inside.

Today's a new day in a new month, which means it's time to talk about new GL stuff. I'm allowed to do that once in a while, even though GL stuff is never actually new. In this post we're going to be looking at [GL_NV_timeline_semaphore](https://registry.khronos.org/OpenGL/extensions/NV/NV_timeline_semaphore.txt), an extension everyone has definitely heard of.

Mesa has supported [GL_EXT_external_objects](https://registry.khronos.org/OpenGL/extensions/EXT/EXT_external_objects.txt) for a long while, and it's no exaggeration to say that this is the industry-standard implementation: there are no proprietary drivers I'm aware of which can pass the super-strict piglit tests we've accumulated over the years. Yes, that includes Green Triangle.

https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/35866
