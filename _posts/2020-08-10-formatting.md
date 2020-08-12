---
published: true
---
## enum pipe_format
One thing that's everywhere in mesa (at least outside of mesa core) is `enum pipe_format`. This enum is used to describe image formats. The general way that it works is that there's a hook in `struct pipe_screen`:
```c
   /**
    * Check if the given pipe_format is supported as a texture or
    * drawing surface.
    * \param bindings  bitmask of PIPE_BIND_*
    */
   bool (*is_format_supported)( struct pipe_screen *,
                                enum pipe_format format,
                                enum pipe_texture_target target,
                                unsigned sample_count,
                                unsigned storage_sample_count,
                                unsigned bindings );
```
Each gallium driver implements this hook, and then gallium queries the driver before creating a resource using a given format and then also before using a created resource in various ways.

# PIPE_BIND
Of particular note in this hook is the `bindings` parameter, which specifies the way(s) that the resource will be used. These values are defined as:
```c
/**
 * Resource binding flags -- gallium frontends must specify in advance all
 * the ways a resource might be used.
 */
#define PIPE_BIND_DEPTH_STENCIL        (1 << 0) /* create_surface */
#define PIPE_BIND_RENDER_TARGET        (1 << 1) /* create_surface */
#define PIPE_BIND_BLENDABLE            (1 << 2) /* create_surface */
#define PIPE_BIND_SAMPLER_VIEW         (1 << 3) /* create_sampler_view */
#define PIPE_BIND_VERTEX_BUFFER        (1 << 4) /* set_vertex_buffers */
#define PIPE_BIND_INDEX_BUFFER         (1 << 5) /* draw_elements */
#define PIPE_BIND_CONSTANT_BUFFER      (1 << 6) /* set_constant_buffer */
#define PIPE_BIND_DISPLAY_TARGET       (1 << 7) /* flush_front_buffer */
/* gap */
#define PIPE_BIND_STREAM_OUTPUT        (1 << 10) /* set_stream_output_buffers */
#define PIPE_BIND_CURSOR               (1 << 11) /* mouse cursor */
#define PIPE_BIND_CUSTOM               (1 << 12) /* gallium frontend/winsys usages */
#define PIPE_BIND_GLOBAL               (1 << 13) /* set_global_binding */
#define PIPE_BIND_SHADER_BUFFER        (1 << 14) /* set_shader_buffers */
#define PIPE_BIND_SHADER_IMAGE         (1 << 15) /* set_shader_images */
#define PIPE_BIND_COMPUTE_RESOURCE     (1 << 16) /* set_compute_resources */
#define PIPE_BIND_COMMAND_ARGS_BUFFER  (1 << 17) /* pipe_draw_info.indirect */
#define PIPE_BIND_QUERY_BUFFER         (1 << 18) /* get_query_result_resource */
```
Each of these correspond to types of usages, with the hook that the created resource is likely to pass through in the comment.

# Problem
As always, this is a problem for zink, mainly because the usage passed before resource creation [doesn't always match the actual usage](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3404). For example, a resource created with just `PIPE_BIND_SAMPLER_VIEW` may eventually use `PIPE_BIND_RENDER_TARGET` and vice versa.

This is very common, as `u_blitter` uses `PIPE_BIND_SAMPLER_VIEW` for blitting, and also many piglit tests will draw an image with `PIPE_BIND_SAMPLER_VIEW` and then use it as a framebuffer attachment, aka `PIPE_BIND_RENDER_TARGET`.

The problem here is that Vulkan is a very specific API, and so at no point is it possible to "add" usage capabilities later. Furthermore, it's also the case that drivers may support either color attachment **or** sampler usage for a given format but not both.

There's two specific parts of zink that this affects:
* resource creation, where the usage bits need to be set for the right usages so that the underlying driver allocates the right memory
* blitting/copying, where it's necessary to know the underlying driver's capabilities for a format in order to be able to choose the right method for the transfer

## Solutions-ish
For the first item, the solution is simple-ish:
```c
/* apparently gallium thinks this is the jack-of-all-trades bind type */
if (templ->bind & PIPE_BIND_SAMPLER_VIEW)
   bci.usage |= VK_BUFFER_USAGE_UNIFORM_TEXEL_BUFFER_BIT |
                VK_BUFFER_USAGE_STORAGE_TEXEL_BUFFER_BIT |
                VK_BUFFER_USAGE_STORAGE_BUFFER_BIT |
                VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT |
                VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT |
                VK_BUFFER_USAGE_INDEX_BUFFER_BIT |
                VK_BUFFER_USAGE_VERTEX_BUFFER_BIT |
                VK_BUFFER_USAGE_TRANSFORM_FEEDBACK_BUFFER_BIT_EXT;
```
Yes, for this case the simple fix is to just set every possible usage bit. Is it the "most correct" way? Maybe. But gallium just doesn't give enough information at this point due to deficiencies in the OpenGL spec which allow objects to sort of just be used for whatever.

For the second item, things get messy.

Some gallium drivers (e.g., iris) check for 3-component (i.e., RGB instead of RGBA) with `PIPE_BIND_SAMPLER_VIEW` and then claim no support so that gallium will provide a 4-component image. This sort of works for zink, except that then there's an extra component and so various Vulkan codepaths are now receiving an extra n-bits per block, which breaks everything.

I haven't really dug into the issue any further than this, but it's an interesting problem, so I thought I'd blog about it.
