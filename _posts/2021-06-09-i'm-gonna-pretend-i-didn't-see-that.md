---
published: true
---
## Memes

We've all been there. No matter how 10x someone is or feels, everyone has had a moment where abruptly they say to themselves, **HOW THE FUCK DO THREADS EVEN WORK?**

This may be precipitated by any number of events, including, but not limited to:
* forgetting a lock
* forgetting to unlock
* missing an unlock at an early return
* forgetting to initialize a lock
* forgetting to spawn a thread
* forgetting to signal a conditional
* forgetting to initialize a conditional
* running the test case with the wrong driver

I'm not going to say that I've been there recently.

I'm not going to say that it was today, nor am I going to state, on the record, that at least one existing zink-wip snapshot may or may not be affected by an issue which may or may not be on the above list.

I'm not going to say any of these things.

What I am going to do is talk about a new oom handler I've been working on to handle the dreaded `spec@!opengl 1.1@streaming-texture-leak` case from piglit.

## The Case
This test is annoying in that it is effectively a test of a driver's ability to throttle itself when an app is generating and using $infinity textures without ever explicitly triggering a flush.

In short, it's:
```c
for (i = 0; i < 5000; i++) {
   glGenTextures(1, &texture);
   glBindTexture(GL_TEXTURE_2D, texture);
   glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, TEX_SIZE, TEX_SIZE, 0, GL_RGBA, GL_UNSIGNED_BYTE, tex_buffer);
   piglit_draw_rect_tex(0, 0, piglit_width, piglit_height, 0, 0, 1, 1);
   glDeleteTextures(1, &texture);
}
```

The textures are "deleted", yes, but because they're in use, the driver can't actually delete them at this point of call, meaning that they can only truly be deleted once they are no longer in use by the GPU. At some iteration, this will begin to oom the GPU, and the driver will have to determine how to handle things.

## The Zink Case
At present, mainline zink uses a hammer-and-nail methodology that I came up with last year: the total amount of GPU memory in use by resources in a given cmdbuf is tracked, and that amount is tracked per-context. If the in-use context memory exceeds a threshold of the total VRAM, the driver stalls, thereby freeing up all the resources that are in use so they can be recycled into new ones.

There's a number of problems with this approach, but the biggest one is that it fails to account for cases like a AAA game that just uses as much memory as it can in order to optimize performance/resolution/graphics. I discovered such a case some time ago while running Tomb Raider, and then I set out to improve things since it was costing me about 10% of my perf on the title screen.

The annoying part of this problem is that the piglit test is a very uncommon case, and it's tricky to handle it in a way that doesn't also impact other cases which appear similar but need to not get memory-clamped. As a result, it's tough to really do anything based on "overall" memory usage.

In the end, what I decided on was using the per-cmdbuf memory usage counter to trigger a check for completed cmdbufs on submit, iterating over all the pending ones to check whether they've completed, resetting them and freeing associated resources when possible. This yields good memory reclaiming behavior for problem cases while leaving games like Tomb Raider untouched and definitely not deadlocking or anything like that.
