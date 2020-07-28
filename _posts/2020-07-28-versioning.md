---
published: false
---
## How Do Mesa Versions Work?

Today I thought it might be interesting to dive into how mesa detects version support for drivers.

To do so, I'm going to be jumping into [mesa/src/mesa/main/version.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/master/src/mesa/main/version.c), which is where the magic happens.

# mesa/main/version.c
In this file are a couple functions which check driver extension support. For the (current) purposes of my zink work, I'll be looking at `compute_version()`, the function handling desktop GL. This function takes a pointer to the extensions as well as the constant values set by the driver, and each GL version is determined by support for the required extensions.

As an example, here's GL 3.0, which is what zink is currently detected as:
```c
   const bool ver_3_0 = (ver_2_1 &&
                         consts->GLSLVersion >= 130 &&
                         (consts->MaxSamples >= 4 || consts->FakeSWMSAA) &&
                         (api == API_OPENGL_CORE ||
                          extensions->ARB_color_buffer_float) &&
                         extensions->ARB_depth_buffer_float &&
                         extensions->ARB_half_float_vertex &&
                         extensions->ARB_map_buffer_range &&
                         extensions->ARB_shader_texture_lod &&
                         extensions->ARB_texture_float &&
                         extensions->ARB_texture_rg &&
                         extensions->ARB_texture_compression_rgtc &&
                         extensions->EXT_draw_buffers2 &&
                         extensions->ARB_framebuffer_object &&
                         extensions->EXT_framebuffer_sRGB &&
                         extensions->EXT_packed_float &&
                         extensions->EXT_texture_array &&
                         extensions->EXT_texture_shared_exponent &&
                         extensions->EXT_transform_feedback &&
                         extensions->NV_conditional_render);
```
Each member of `extensions` being checked is the full name of an extension, so it can easily be found by a search. Each member of `consts` is a constant value exported by the driver. Most notable is `GLSLVersion`, which increases for every GL version beginning with 3.0.

Thus, saying "zink supports GL 3.0" is really equivalent to passing this check.

But wait, this is mesa core, and zink is a gallium driver.

# mesa/state_tracker/st_extensions.c
Midway through this file is [st_init_extensions()](https://gitlab.freedesktop.org/mesa/mesa/-/blob/master/src/mesa/state_tracker/st_extensions.c) (direct line link omitted for future compatibility). Here, `enum pipe_cap` is mapped to extension support, but this is only for some cases:
```c
   static const struct st_extension_cap_mapping cap_mapping[] = {
      { o(ARB_base_instance),                PIPE_CAP_START_INSTANCE                   },
      { o(ARB_bindless_texture),             PIPE_CAP_BINDLESS_TEXTURE                 },
      { o(ARB_buffer_storage),               PIPE_CAP_BUFFER_MAP_PERSISTENT_COHERENT   },
      { o(ARB_clear_texture),                PIPE_CAP_CLEAR_TEXTURE                    },
      { o(ARB_clip_control),                 PIPE_CAP_CLIP_HALFZ                       },
      { o(ARB_color_buffer_float),           PIPE_CAP_VERTEX_COLOR_UNCLAMPED           },
      { o(ARB_conditional_render_inverted),  PIPE_CAP_CONDITIONAL_RENDER_INVERTED      },
      { o(ARB_copy_image),                   PIPE_CAP_COPY_BETWEEN_COMPRESSED_AND_PLAIN_FORMATS },
```
Setting a nonzero value for these pipe caps enable the corresponding extension for version checking.

Other extensions are determined by values of pipe caps later on:
```c
   if (GLSLVersion >= 400 && !options->disable_arb_gpu_shader5)
      extensions->ARB_gpu_shader5 = GL_TRUE;
```
In this case, supporting at least GLSL 4.00 and not explicitly disabling the extension is enough to enable it.

In this way, there's no method to do "enable GL $version", and everything is instead inferred based on the driver's capabilities, letting us remain reasonably confident that drivers really do support all parts of every GL version they claim to support.