---
published: true
---
## Directing Indirection

Now on a wildly different topic, I'm going to talk about [indirect drawing](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_draw_indirect.txt) for a bit, specifically when using it in combination with [primitive restart](https://www.khronos.org/registry/OpenGL/extensions/NV/NV_primitive_restart.txt), which I already briefly talked about in a prior post.

In general, indirect drawing is used when an application wants to provide the gpu with a buffer containing the parameters to be used for draw calls. The idea is that the parameters are already "on the CPU", so there's no back-and-forth needed with the CPU for cases where these parameters may be derived in the course of GPU operations.

The problem here for zink is that `indirect drawing` can be used with `primitive restart`, but the problem I brought up previously still exists, namely that OpenGL allows arbitrary values for the restart index, whereas Vulkan requires a fixed value.

This means that in order for indirect draws to work correctly with primitive restart after being translated to Vulkan, zink needs to...
![harold.jpg]({{site.url}}/assets/harold.jpg)

Yes, zink needs to map those buffers used for indirect drawing and rewrite the indices to convert the arbitrary restart index to the fixed one.

## Utils, Utils, Utils
Thankfully, all of this can be done in the utility functions in `u_prim_restart.c` that I talked about in the primitive restart post, which provide functionality for both rewriting index buffers to perform restart index conversion as well as handling draw calls for unsupported primitive types.

The `ARB_draw_indirect` spec defines two buffer formats for use with indirect draws:
```c
typedef struct {
  GLuint count;
  GLuint primCount;
  GLuint first;
  GLuint reservedMustBeZero;
} DrawArraysIndirectCommand;

typedef struct {
  GLuint count;
  GLuint primCount;
  GLuint firstIndex;
  GLint  baseVertex;
  GLuint reservedMustBeZero;
} DrawElementsIndirectCommand;
```
One is for array draws, the other for elements. Happily, only the first three members are of use for this awfulness, so there's no need to determine which type of draw it is before grabbing that buffer with both hands and telling the GPU to completely stop everything that it wanted to do so we can push up our reading glasses and take a nice, slow read.

With the buffer contents in hand and the GPU performance having dropped to a nonexistent state, the indirect draw command can then be rewritten as a direct draw since the buffer is already mapped, and the entire premise of the indirect draw can be invalidated for the sake of compatibility.

For those interested in seeing what this looks like, the MR is [here](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5886).
