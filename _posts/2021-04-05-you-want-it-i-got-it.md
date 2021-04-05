---
published: false
---
## Woosh

After last week's post touting the "final" features being added to the upcoming Mesa release, naturally now that this is a new week, I have to outdo myself.

I've heard some speculation about zink's future regarding features. Specifically regarding all the [mesamatrix](https://mesamatrix.net/) features that aren't green-ified for zink yet.

So you want features, is what you're saying.

Let's see where things stand after the weekend:
* **GL_OES_tessellation_shader**, **GL_OES_gpu_shader5** - this is a [mesamatrix bug](https://github.com/MightyCreak/mesamatrix/issues/193); zink can't reach GL 4.0 without supporting them, so obviously they are supported
* **GL_ARB_bindless_texture** - the final boss
* **GL_ARB_cl_event** - not (yet) supported by mesa
* **GL_ARB_compute_variable_group_size** - done
* **GL_ARB_ES3_2_compatibility** - missing advanced blend from ES3.2
* **GL_ARB_fragment_shader_interlock** - [done](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/10013)
* **GL_ARB_gpu_shader_int64** - done
* **GL_ARB_parallel_shader_compile** - done
* **GL_ARB_post_depth_coverage** - done (thanks ajax)
* **GL_ARB_robustness_isolation** - not supported by mesa
* **GL_ARB_sample_locations** - [done](https://gitlab.freedesktop.org/zmike/mesa/-/commit/769e946a8edb8912caf997ba217ff740bc4b6169)
* **GL_ARB_seamless_cubemap_per_texture** - needs a new Vulkan extension
* **GL_ARB_shader_ballot** - [done](https://gitlab.freedesktop.org/zmike/mesa/-/commit/82d21eae0a15838cd5f06e11937ff06a8fcc1d5f)
* **GL_ARB_shader_clock** - [done](https://gitlab.freedesktop.org/zmike/mesa/-/commit/4baa239aebc001b253fb8d55e80ab8e88d1df066)
* **GL_ARB_shader_stencil_export** - done
* **GL_ARB_shader_viewport_layer_array** - done
* **GL_ARB_shading_language_include** - done
* **GL_ARB_sparse_buffer** - [done](https://gitlab.freedesktop.org/zmike/mesa/-/commit/471e82c20c1720eda613619c2257d6d5ce949e4b)
* **GL_ARB_sparse_texture** - not supported by mesa
* **GL_ARB_sparse_texture2** - not supported by mesa
* **GL_ARB_sparse_texture_clamp** - not supported by mesa
* **GL_ARB_texture_filter_minmax** - [done](https://gitlab.freedesktop.org/zmike/mesa/-/commit/ae8e926fd60b35c929eb52af8a11a0eefbac4605)
* **GL_EXT_memory_object** - TODO
* **GL_EXT_memory_object_fd** - TODO
* **GL_EXT_memory_object_win32** - not supported by mesa
* **GL_EXT_render_snorm** - done
* **GL_EXT_semaphore** - TODO
* **GL_EXT_semaphore_fd** - TODO
* **GL_EXT_semaphore_win32** - not supported by mesa
* **GL_EXT_sRGB_write_control** - TODO
* **GL_EXT_texture_norm16** - done
* **GL_EXT_texture_sRGB_R8** - TODO
* **GL_KHR_blend_equation_advanced_coherent** - same as regular advanced blend
* **GL_KHR_texture_compression_astc_hdr** - TODO
* **GL_KHR_texture_compression_astc_sliced_3d** - TODO
* **GL_OES_depth_texture_cube_map** - done
* **GL_OES_EGL_image** - done
* **GL_OES_EGL_image_external** - done
* **GL_OES_EGL_image_external_essl3** - done
* **GL_OES_required_internalformat** - done
* **GL_OES_surfaceless_context** - done
* **GL_OES_texture_compression_astc** - TODO
* **GL_OES_texture_float** - done
* **GL_OES_texture_float_linear** - done
* **GL_OES_texture_half_float** - done
* **GL_OES_texture_half_float_linear** - done
* **GL_OES_texture_view** - same [mesamatrix bug](https://github.com/MightyCreak/mesamatrix/issues/193) since this is a GL 4.3 extension
* **GL_OES_viewport_array** - done
* **GLX_ARB_context_flush_control** - not supported by mesa
* **GLX_ARB_robustness_application_isolation** - not supported by mesa
* **GLX_ARB_robustness_share_group_isolation** - not supported by mesa
* **GL_EXT_shader_group_vote** - done
* **GL_EXT_multisampled_render_to_texture** - TODO
* **GL_EXT_color_buffer_half_float¶** - TODO
* **GL_EXT_depth_bounds_test** - done

By my calculations, that's 11 `TODO`s, 10 `not supported`s, 2 `advanced blend`, and 1 `final boss`, a total of 24 out-of-version-extensions not yet implemented out of 54, meaning that 30 are done, tying with i965 and second only to RadeonSI at 33.


