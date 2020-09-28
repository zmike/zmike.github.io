---
published: false
---
## But No, I'm not

Just a quick post today to summarize a few exciting changes I've made today.

To start with, I've added some tracking to the internal batch objects for catching things like piglit's `spec@!opengl 1.1@streaming-texture-leak`. Let's check out the test code there for a moment:

```c
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


[![epilogue.png]({{site.url}}/assets/bench1/endpost1.png)]({{site.url}}/assets/bench1/epilogue.png)