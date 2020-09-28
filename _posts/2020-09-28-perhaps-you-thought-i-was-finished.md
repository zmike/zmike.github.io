---
published: true
---
## But No, I'm not

Just a quick post today to summarize a few exciting changes I've made today.

To start with, I've added some tracking to the internal batch objects for catching things like piglit's `spec@!opengl 1.1@streaming-texture-leak`. Let's check out the test code there for a moment:

```c
/** @file streaming-texture-leak.c
 *
 * Tests that allocating and freeing textures over and over doesn't OOM
 * the system due to various refcounting issues drivers may have.
 *
 * Textures used are around 4MB, and we make 5k of them, so OOM-killer
 * should catch any failure.
 *
 * Bug #23530
 */
for (i = 0; i < 5000; i++) {
        glGenTextures(1, &texture);
        glBindTexture(GL_TEXTURE_2D, texture);

        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER,
                        GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER,
                        GL_LINEAR);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, TEX_SIZE, TEX_SIZE,
                     0, GL_RGBA,
                     GL_UNSIGNED_BYTE, tex_buffer);

        piglit_draw_rect_tex(0, 0, piglit_width, piglit_height,
                             0, 0, 1, 1);

        glDeleteTextures(1, &texture);
}
```
This test loops 5000 times, using a different sampler texture for each draw, and then destroys the texture. This is supposed to catch drivers which can't properly manage their resource refcounts, but instead here zink is getting caught by trying to dump 5000 active resources into the same command buffer, which ooms the system.

The reason for this is that, after my recent optimizations which avoid unnecessary flushing, zink only submits the draw command when a frame is finished or one of the write-flagged resources associated with an active batch is read from. Thus, the whole test runs in one go, only submitting the queue at the very end when the test performs a read.

In this case, my fix is simple: check the system's total memory on driver init, and then always flush a batch if it crosses some threshold of memory usage in its associated resources when beginning a new draw. I chose 1/8 total memory to be "safe", since that allows zink to use 50% of the total memory with its resources before it'll begin to stall and force the draws to complete, hopefully avoiding any oom scenarios. This ends up being a flush every 250ish draws in the above test code, and everything works nicely without killing my system.


## Performance 3.0
As a bonus, I noticed that zink was taking considerably longer than IRIS to complete this test once it was fixed, so I did a little profiling, and this was the result:
[![epilogue.png]({{site.url}}/assets/bench1/endpost1.png)]({{site.url}}/assets/bench1/epilogue.png)

Up another 3 fps (~10%) from Friday, which isn't bad for a few minutes spent removing some `memset` calls from descriptor updating and then throwing in some code for handling [VK_DYNAMIC_STATE_VERTEX_INPUT_BINDING_STRIDE_EXT](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_extended_dynamic_state.html).
