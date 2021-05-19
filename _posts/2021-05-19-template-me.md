---
published: false
---
## Using Power For Evil

There's no shortage of very smart people working on Mesa. One of those, aspiring benchmark-quadrupler Marek Olšák, had a novel idea some time ago: Could C++ function templates were used to optimize draw dispatch in driver?

The answer was yes, and so began what was probably five or ten minutes of furiously jamming brackets and braces into [a C++ file](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-21.1.0/src/gallium/drivers/radeonsi/si_state_draw.cpp) in order to achieve the intended result. Let's check out what's going on here.

## Setup
To start, the templates must be accessible from C, as this is what the driver is written in. The methodology here is simple: Generate the templates as an array of function pointers such that they can be accessed by indexing the arrays with the template values. Here's what the code looks like:

```c
template <chip_class GFX_VERSION, si_has_tess HAS_TESS, si_has_gs HAS_GS,
          si_has_ngg NGG, si_has_prim_discard_cs ALLOW_PRIM_DISCARD_CS>
static void si_init_draw_vbo(struct si_context *sctx)
{
   /* Prim discard CS is only useful on gfx7+ because gfx6 doesn't have async compute. */
   if (ALLOW_PRIM_DISCARD_CS && GFX_VERSION < GFX7)
      return;

   if (NGG && GFX_VERSION < GFX10)
      return;

   sctx->draw_vbo[GFX_VERSION - GFX6][HAS_TESS][HAS_GS][NGG][ALLOW_PRIM_DISCARD_CS] =
      si_draw_vbo<GFX_VERSION, HAS_TESS, HAS_GS, NGG, ALLOW_PRIM_DISCARD_CS>;
}

template <chip_class GFX_VERSION, si_has_tess HAS_TESS, si_has_gs HAS_GS>
static void si_init_draw_vbo_all_internal_options(struct si_context *sctx)
{
   si_init_draw_vbo<GFX_VERSION, HAS_TESS, HAS_GS, NGG_OFF, PRIM_DISCARD_CS_OFF>(sctx);
   si_init_draw_vbo<GFX_VERSION, HAS_TESS, HAS_GS, NGG_OFF, PRIM_DISCARD_CS_ON>(sctx);
   si_init_draw_vbo<GFX_VERSION, HAS_TESS, HAS_GS, NGG_ON, PRIM_DISCARD_CS_OFF>(sctx);
   si_init_draw_vbo<GFX_VERSION, HAS_TESS, HAS_GS, NGG_ON, PRIM_DISCARD_CS_ON>(sctx);
}

template <chip_class GFX_VERSION>
static void si_init_draw_vbo_all_pipeline_options(struct si_context *sctx)
{
   si_init_draw_vbo_all_internal_options<GFX_VERSION, TESS_OFF, GS_OFF>(sctx);
   si_init_draw_vbo_all_internal_options<GFX_VERSION, TESS_OFF, GS_ON>(sctx);
   si_init_draw_vbo_all_internal_options<GFX_VERSION, TESS_ON, GS_OFF>(sctx);
   si_init_draw_vbo_all_internal_options<GFX_VERSION, TESS_ON, GS_ON>(sctx);
}

static void si_init_draw_vbo_all_families(struct si_context *sctx)
{
   si_init_draw_vbo_all_pipeline_options<GFX6>(sctx);
   si_init_draw_vbo_all_pipeline_options<GFX7>(sctx);
   si_init_draw_vbo_all_pipeline_options<GFX8>(sctx);
   si_init_draw_vbo_all_pipeline_options<GFX9>(sctx);
   si_init_draw_vbo_all_pipeline_options<GFX10>(sctx);
   si_init_draw_vbo_all_pipeline_options<GFX10_3>(sctx);
}

static void si_invalid_draw_vbo(struct pipe_context *pipe,
                                const struct pipe_draw_info *info,
                                const struct pipe_draw_indirect_info *indirect,
                                const struct pipe_draw_start_count *draws,
                                unsigned num_draws)
{
   unreachable("vertex shader not bound");
}

extern "C"
void si_init_draw_functions(struct si_context *sctx)
{
   si_init_draw_vbo_all_families(sctx);

   /* Bind a fake draw_vbo, so that draw_vbo isn't NULL, which would skip
    * initialization of callbacks in upper layers (such as u_threaded_context).
    */
   sctx->b.draw_vbo = si_invalid_draw_vbo;
   sctx->blitter->draw_rectangle = si_draw_rectangle;

   si_init_ia_multi_vgt_param_table(sctx);
}
```
This calls through a series of functions, ultimately reaching `si_init_draw_vbo` where a template is set to a member of the function pointer array based on the template parameters. Specialized functions can then be generated based on hardware type, pipeline shader presence, and more.

## Application
Once initialized, there's an inline function used to set the current function pointer:

```c
static inline void si_select_draw_vbo(struct si_context *sctx)
{
   sctx->b.draw_vbo = sctx->draw_vbo[sctx->chip_class - GFX6]
                                    [!!sctx->shader.tes.cso]
                                    [!!sctx->shader.gs.cso]
                                    [sctx->ngg]
                                    [si_compute_prim_discard_enabled(sctx)];
   assert(sctx->b.draw_vbo);
}
```
Thus the parameters are pulled directly from the context, and the function can be called whenever the draw function pointer needs to be updated, such as when new shaders are bound or primitive discard is enabled.

## Result
The result is that now the draw dispatch can be fully optimized for the codepath required by the active hardware and graphics pipeline, reducing the CPU overhead and making the draw code the tiniest bit faster. For example, here's just the top part of the templated function:

```c
template <chip_class GFX_VERSION, si_has_tess HAS_TESS, si_has_gs HAS_GS, si_has_ngg NGG,
          si_has_prim_discard_cs ALLOW_PRIM_DISCARD_CS>
static void si_draw_vbo(struct pipe_context *ctx,
                        const struct pipe_draw_info *info,
                        unsigned drawid_offset,
                        const struct pipe_draw_indirect_info *indirect,
                        const struct pipe_draw_start_count_bias *draws,
                        unsigned num_draws)
{
   /* Keep code that uses the least number of local variables as close to the beginning
    * of this function as possible to minimize register pressure.
    *
    * It doesn't matter where we return due to invalid parameters because such cases
    * shouldn't occur in practice.
    */
   struct si_context *sctx = (struct si_context *)ctx;

   /* Recompute and re-emit the texture resource states if needed. */
   unsigned dirty_tex_counter = p_atomic_read(&sctx->screen->dirty_tex_counter);
   if (unlikely(dirty_tex_counter != sctx->last_dirty_tex_counter)) {
      sctx->last_dirty_tex_counter = dirty_tex_counter;
      sctx->framebuffer.dirty_cbufs |= ((1 << sctx->framebuffer.state.nr_cbufs) - 1);
      sctx->framebuffer.dirty_zsbuf = true;
      si_mark_atom_dirty(sctx, &sctx->atoms.s.framebuffer);
      si_update_all_texture_descriptors(sctx);
   }

   unsigned dirty_buf_counter = p_atomic_read(&sctx->screen->dirty_buf_counter);
   if (unlikely(dirty_buf_counter != sctx->last_dirty_buf_counter)) {
      sctx->last_dirty_buf_counter = dirty_buf_counter;
      /* Rebind all buffers unconditionally. */
      si_rebind_buffer(sctx, NULL);
   }

   si_decompress_textures(sctx, u_bit_consecutive(0, SI_NUM_GRAPHICS_SHADERS));
   si_need_gfx_cs_space(sctx, num_draws);

   /* If we're using a secure context, determine if cs must be secure or not */
   if (GFX_VERSION >= GFX9 && unlikely(radeon_uses_secure_bos(sctx->ws))) {
      bool secure = si_gfx_resources_check_encrypted(sctx);
      if (secure != sctx->ws->cs_is_secure(&sctx->gfx_cs)) {
         si_flush_gfx_cs(sctx, RADEON_FLUSH_ASYNC_START_NEXT_GFX_IB_NOW |
                               RADEON_FLUSH_TOGGLE_SECURE_SUBMISSION, NULL);
      }
   }

   if (HAS_TESS) {
      struct si_shader_selector *tcs = sctx->shader.tcs.cso;

      /* The rarely occuring tcs == NULL case is not optimized. */
      bool same_patch_vertices =
         GFX_VERSION >= GFX9 &&
         tcs && info->vertices_per_patch == tcs->info.base.tess.tcs_vertices_out;

      if (sctx->same_patch_vertices != same_patch_vertices) {
         sctx->same_patch_vertices = same_patch_vertices;
         sctx->do_update_shaders = true;
      }

      if (GFX_VERSION == GFX9 && sctx->screen->info.has_ls_vgpr_init_bug) {
         /* Determine whether the LS VGPR fix should be applied.
          *
          * It is only required when num input CPs > num output CPs,
          * which cannot happen with the fixed function TCS. We should
          * also update this bit when switching from TCS to fixed
          * function TCS.
          */
         bool ls_vgpr_fix =
            tcs && info->vertices_per_patch > tcs->info.base.tess.tcs_vertices_out;

         if (ls_vgpr_fix != sctx->ls_vgpr_fix) {
            sctx->ls_vgpr_fix = ls_vgpr_fix;
            sctx->do_update_shaders = true;
         }
      }
```
Note that the hardware version parts are templated, as is the `HAS_TESS` conditional, enabling it to be skipped entirely if there's no tessellation shader active.