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
