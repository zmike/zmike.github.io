---
published: false
---
## The Optimizations Continue
[Optimizing transfer_map](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2966) is one of the first issues I created, and it's definitely one of the most important, at least as it pertains to unit tests. So many unit tests perform reads on buffers that it's crucial to ensure no unnecessary flushing or stalling is happening here.

Today, I've made further strides in this direction for piglit's *spec@glsl-1.30@execution@texelfetch fs sampler2d 1x281-501x281*:

**before**

`MESA_LOADER_DRIVER_OVERRIDE=zink bin/texelFetch   4.41s user 1.92s system 71% cpu 8.801 total`

**after**

`MESA_LOADER_DRIVER_OVERRIDE=zink bin/texelFetch   4.22s user 1.72s system 76% cpu 7.749 total`

## More Speed Loops
As part of ensuring test coherency, a lot of explicit fencing was added around `transfer_map` and `transfer_unmap`. Part of this was to work around the lack of good barrier usage, and part of it was just to make sure that everything was properly synchronized.

An especially non-performant case of this was in `transfer_unmap`, where I added a fence to block after the buffer was unmapped. In reality, this makes no sense other than for the case of synchronizing with the compute batch, which is a bit detached from everything else.

The reason this fence continued to be needed comes down to barrier usage for descriptors. At the start, all image resources used for descriptors just used `VK_IMAGE_LAYOUT_GENERAL` for the barrier layout, which is fine and good; certainly this is spec compliant. It isn't, however, optimally informing the underlying driver about the usage for the case where the resource was previously used with `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` to copy data back from a staging image.

Instead, `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` can be used for sampler images since they're read-only. This differs from `VK_IMAGE_LAYOUT_GENERAL` in that `VK_IMAGE_LAYOUT_GENERAL` contains both read and write—not what's actually going on in this case given that sampler images are read-only.



