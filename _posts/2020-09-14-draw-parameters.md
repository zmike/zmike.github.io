---
published: true
---
## Hoo Boy

Let's talk about [ARB_shader_draw_parameters](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_shader_draw_parameters.txt). Specifically, let's look at [gl_BaseVertex](https://www.khronos.org/opengl/wiki/Vertex_Shader/Defined_Inputs).

In OpenGL, this shader variable's value depends on the parameters passed to the draw command, and the value is always zero if the command has no base vertex.

In Vulkan, the value here is only zero if the first vertex is zero.

The difference here means that for arrayed draws without base vertex parameters, GL always expects zero, and Vulkan expects `first vertex`.

Hooray.

## Compatibilizing

The easiest solution here would be to just throw a shader key at the problem, producing variants of the shader for use with indexed vs non-indexed draws, and using NIR passes to modify the variables for the non-indexed case and zero the value. It's quick, it's easy, and it's not especially great for performance since it requires compiling the shader multiple times and creating multiple pipeline objects.

This is where push constants come in handy once more.

Avid readers of the blog will recall the last time I used push constants was for TCS injection when I needed to generate my own TCS and have it read the default inner/outer tessellation levels out of a push constant.

Since then, I've created a struct to track the layout of the push constant:
```c
struct zink_push_constant {
   unsigned draw_mode_is_indexed;
   float default_inner_level[2];
   float default_outer_level[4];
};
```
Now just before draw, I update the push constant value for `draw_mode_is_indexed`:
```c
if (ctx->gfx_stages[PIPE_SHADER_VERTEX]->nir->info.system_values_read & (1ull << SYSTEM_VALUE_BASE_VERTEX)) {
   unsigned draw_mode_is_indexed = dinfo->index_size > 0;
   vkCmdPushConstants(batch->cmdbuf, gfx_program->layout, VK_SHADER_STAGE_VERTEX_BIT,
                      offsetof(struct zink_push_constant, draw_mode_is_indexed), sizeof(unsigned),
                      &draw_mode_is_indexed);
}
```
And now the shader can be made aware of whether the draw mode is indexed.

Now comes the NIR, as is the case for most of this type of work.

```c
static bool
lower_draw_params(nir_shader *shader)
{
   if (shader->info.stage != MESA_SHADER_VERTEX)
      return false;

   if (!(shader->info.system_values_read & (1ull << SYSTEM_VALUE_BASE_VERTEX)))
      return false;

   return nir_shader_instructions_pass(shader, lower_draw_params_instr, nir_metadata_dominance, NULL);
}
```
This is the future, so I'm now using Eric Anholt's [recent helper function](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/6412) to skip past iterating over the shader's function/blocks/instructions, instead just passing the lowering implementation as a parameter and letting the helper create the `nir_builder` for me.

```c
static bool
lower_draw_params_instr(nir_builder *b, nir_instr *in, void *data)
{
   if (in->type != nir_instr_type_intrinsic)
      return false;
   nir_intrinsic_instr *instr = nir_instr_as_intrinsic(in);
   if (instr->intrinsic != nir_intrinsic_load_base_vertex)
      return false;
```
I'm filtering out everything except for `nir_intrinsic_load_base_vertex` here, which is the instruction for loading `gl_BaseVertex`.
```c
   
   b->cursor = nir_after_instr(&instr->instr);
```
I'm modifying instructions after this one, so I set the cursor after.
```c
   nir_intrinsic_instr *load = nir_intrinsic_instr_create(b->shader, nir_intrinsic_load_push_constant);
   load->src[0] = nir_src_for_ssa(nir_imm_int(b, 0));
   nir_intrinsic_set_range(load, 4);
   load->num_components = 1;
   nir_ssa_dest_init(&load->instr, &load->dest, 1, 32, "draw_mode_is_indexed");
   nir_builder_instr_insert(b, &load->instr);
```
I'm loading the first 4 bytes of the push constant variable that I created according to my struct, which is the `draw_mode_is_indexed` value.
```c
   nir_ssa_def *composite = nir_build_alu(b, nir_op_bcsel,
                                          nir_build_alu(b, nir_op_ieq, &load->dest.ssa, nir_imm_int(b, 1), NULL, NULL),
                                          &instr->dest.ssa,
                                          nir_imm_int(b, 0),
                                          NULL);
```
This adds a new ALU instruction of type `bcsel`, AKA the ternary operator (condition ? true : false). The condition here is another ALU of type `ieq`, AKA integer equals, and I'm testing whether the loaded push constant value is equal to 1. If true, this is an indexed draw, so I continue using the loaded `gl_BaseVertex` value. If false, this is not an indexed draw, so I need to use zero instead.
```c
   nir_ssa_def_rewrite_uses_after(&instr->dest.ssa, nir_src_for_ssa(composite), composite->parent_instr);
```
With my `bcsel` composite `gl_BaseVertex` value constructed, I can now rewrite all subsequent uses of `gl_BaseVertex` in the shader to use the composite value, which will automatically swap between the Vulkan `gl_BaseVertex` and zero based on the value of the push constant without the need to rebuild the shader or make a new pipeline.
```c
   return true;
}
```

And now the shader gets the expected value and everything works.

## Billy Mays
It's also worth pointing out here that `gl_DrawID` from the same extension has a similar problem: gallium doesn't pass multidraws in full to the driver, instead iterating for each draw, which means that the shader value is never what's expected either. I've employed a similar trick to jam the draw index into the push constant and read that back in the shader to get the expected value there too.

Extensions.
