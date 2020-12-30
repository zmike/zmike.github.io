---
published: false
---
## A Different Sort Of Optimization

There's a number of strange hacks in zink that provide compatibility for some of the layers in mesa. One of these hacks is the NIR pass used for managing non-constant UBO/SSBO array indexing, made necessary because SPIRV operates by directly accessing variables, and so it's impossible to have a non-constant index because then when generating the SPIRV there's no way to know which variable is being accessed.

In its current state from zink-wip it looks like this:
```c
static nir_ssa_def *
recursive_generate_bo_ssa_def(nir_builder *b, nir_intrinsic_instr *instr, nir_ssa_def *index, unsigned start, unsigned end)
{
   if (start == end - 1) {
      /* block index src is 1 for this op */
      unsigned block_idx = instr->intrinsic == nir_intrinsic_store_ssbo;
      nir_intrinsic_instr *new_instr = nir_intrinsic_instr_create(b->shader, instr->intrinsic);
      new_instr->src[block_idx] = nir_src_for_ssa(nir_imm_int(b, start));
      for (unsigned i = 0; i < nir_intrinsic_infos[instr->intrinsic].num_srcs; i++) {
         if (i != block_idx)
            nir_src_copy(&new_instr->src[i], &instr->src[i], &new_instr->instr);
      }
      if (instr->intrinsic != nir_intrinsic_load_ubo_vec4) {
         nir_intrinsic_set_align(new_instr, nir_intrinsic_align_mul(instr), nir_intrinsic_align_offset(instr));
         if (instr->intrinsic != nir_intrinsic_load_ssbo)
            nir_intrinsic_set_range(new_instr, nir_intrinsic_range(instr));
      }
      new_instr->num_components = instr->num_components;
      if (instr->intrinsic != nir_intrinsic_store_ssbo)
         nir_ssa_dest_init(&new_instr->instr, &new_instr->dest,
                           nir_dest_num_components(instr->dest),
                           nir_dest_bit_size(instr->dest), NULL);
      nir_builder_instr_insert(b, &new_instr->instr);
      return &new_instr->dest.ssa;
   }

   unsigned mid = start + (end - start) / 2;
   return nir_build_alu(b, nir_op_bcsel, nir_build_alu(b, nir_op_ilt, index, nir_imm_int(b, mid), NULL, NULL),
      recursive_generate_bo_ssa_def(b, instr, index, start, mid),
      recursive_generate_bo_ssa_def(b, instr, index, mid, end),
      NULL
   );
}

static bool
lower_dynamic_bo_access_instr(nir_intrinsic_instr *instr, nir_builder *b)
{
   if (instr->intrinsic != nir_intrinsic_load_ubo &&
       instr->intrinsic != nir_intrinsic_load_ubo_vec4 &&
       instr->intrinsic != nir_intrinsic_get_ssbo_size &&
       instr->intrinsic != nir_intrinsic_load_ssbo &&
       instr->intrinsic != nir_intrinsic_store_ssbo)
      return false;
   /* block index src is 1 for this op */
   unsigned block_idx = instr->intrinsic == nir_intrinsic_store_ssbo;
   if (nir_src_is_const(instr->src[block_idx]))
      return false;
   b->cursor = nir_after_instr(&instr->instr);
   bool ssbo_mode = instr->intrinsic != nir_intrinsic_load_ubo && instr->intrinsic != nir_intrinsic_load_ubo_vec4;
   unsigned first_idx = 0, last_idx;
   if (ssbo_mode) {
      last_idx = first_idx + b->shader->info.num_ssbos;
   } else {
      /* skip 0 index if uniform_0 is one we created previously */
      first_idx = !b->shader->info.first_ubo_is_default_ubo;
      last_idx = first_idx + b->shader->info.num_ubos;
   }

   /* now create the composite dest with a bcsel chain based on the original value */
   nir_ssa_def *new_dest = recursive_generate_bo_ssa_def(b, instr,
                                                       instr->src[block_idx].ssa,
                                                       first_idx, last_idx);

   if (instr->intrinsic != nir_intrinsic_store_ssbo)
      /* now use the composite dest in all cases where the original dest (from the dynamic index)
       * was used and remove the dynamically-indexed load_*bo instruction
       */
      nir_ssa_def_rewrite_uses_after(&instr->dest.ssa, nir_src_for_ssa(new_dest), &instr->instr);

   nir_instr_remove(&instr->instr);
   return true;
}
```
In brief, `lower_dynamic_bo_access_instr()` is used to detect UBO/SSBO instructions with a non-constant index, e.g., `array_of_ubos[n]` where `n` is a uniform. Following this, `recursive_generate_bo_ssa_def()` generates a chain of `bcsel` instructions which checks the non-constant array index against constant values and then, upon matching, uses the value loaded from that UBO.

Without going into more depth about the exact mechanics of this pass for the sake of time, I'll instead provide a better explanation by example. Here's a stripped down version of one of the simplest piglit shader tests for non-constant uniform indexing (fs-array-nonconst):

```
[require]
GLSL >= 1.50
GL_ARB_gpu_shader5

[vertex shader passthrough]

[fragment shader]
#version 150
#extension GL_ARB_gpu_shader5: require

uniform block {
	vec4 color[2];
} arr[4];

uniform int n;
uniform int m;

out vec4 color;

void main()
{
	color = arr[n].color[m];
}

[test]
clear color 0.2 0.2 0.2 0.2
clear

ubo array index 0
uniform vec4 block.color[0] 0.0 1.0 1.0 0.0
uniform vec4 block.color[1] 1.0 0.0 0.0 0.0

uniform int n 0
uniform int m 1
draw rect -1 -1 1 1

relative probe rect rgb (0.0, 0.0, 0.5, 0.5) (1.0, 0.0, 0.0)
```
Using two uniforms, a color is indexed from a UBO as the FS output.

In the currently shipping version of zink, the final NIR output from ANV of the fragment shader might look something like this:
```
shader: MESA_SHADER_FRAGMENT
inputs: 0
outputs: 0
uniforms: 8
ubos: 5
shared: 0
decl_var shader_out INTERP_MODE_NONE vec4 color (FRAG_RESULT_DATA0.xyzw, 8, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec1 32 ssa_0 = load_const (0x00000002 /* 0.000000 */)
	vec1 32 ssa_1 = load_const (0x00000001 /* 0.000000 */)
	vec1 32 ssa_2 = load_const (0x00000004 /* 0.000000 */)
	vec1 32 ssa_3 = load_const (0x00000003 /* 0.000000 */)
	vec1 32 ssa_4 = load_const (0x00000010 /* 0.000000 */)
	vec1 32 ssa_5 = intrinsic load_ubo (ssa_1, ssa_4) (0, 1073741824, 16, 0, -1) /* access=0 */ /* align_mul=1073741824 */ /* align_offset=16 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_6 = load_const (0x00000000 /* 0.000000 */)
	vec1 32 ssa_7 = intrinsic load_ubo (ssa_1, ssa_6) (0, 1073741824, 0, 0, -1) /* access=0 */ /* align_mul=1073741824 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_8 = umin ssa_7, ssa_3
	vec1 32 ssa_9 = ishl ssa_5, ssa_2
	vec1 32 ssa_10 = iadd ssa_8, ssa_1
	vec1 32 ssa_11 = load_const (0xfffffffc /* -nan */)
	vec1 32 ssa_12 = iand ssa_9, ssa_11
	vec1 32 ssa_13 = load_const (0x00000005 /* 0.000000 */)
	vec4 32 ssa_14 = intrinsic load_ubo (ssa_13, ssa_12) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec4 32 ssa_23 = intrinsic load_ubo (ssa_2, ssa_12) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_27 = ilt32 ssa_10, ssa_2
	vec1 32 ssa_28 = b32csel ssa_27, ssa_23.x, ssa_14.x
	vec1 32 ssa_29 = b32csel ssa_27, ssa_23.y, ssa_14.y
	vec1 32 ssa_30 = b32csel ssa_27, ssa_23.z, ssa_14.z
	vec1 32 ssa_31 = b32csel ssa_27, ssa_23.w, ssa_14.w
	vec4 32 ssa_32 = intrinsic load_ubo (ssa_3, ssa_12) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_36 = ilt32 ssa_10, ssa_3
	vec1 32 ssa_37 = b32csel ssa_36, ssa_32.x, ssa_28
	vec1 32 ssa_38 = b32csel ssa_36, ssa_32.y, ssa_29
	vec1 32 ssa_39 = b32csel ssa_36, ssa_32.z, ssa_30
	vec1 32 ssa_40 = b32csel ssa_36, ssa_32.w, ssa_31
	vec4 32 ssa_41 = intrinsic load_ubo (ssa_0, ssa_12) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec4 32 ssa_45 = intrinsic load_ubo (ssa_1, ssa_12) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_49 = ilt32 ssa_10, ssa_1
	vec1 32 ssa_50 = b32csel ssa_49, ssa_45.x, ssa_41.x
	vec1 32 ssa_51 = b32csel ssa_49, ssa_45.y, ssa_41.y
	vec1 32 ssa_52 = b32csel ssa_49, ssa_45.z, ssa_41.z
	vec1 32 ssa_53 = b32csel ssa_49, ssa_45.w, ssa_41.w
	vec1 32 ssa_54 = ilt32 ssa_10, ssa_0
	vec1 32 ssa_55 = b32csel ssa_54, ssa_50, ssa_37
	vec1 32 ssa_56 = b32csel ssa_54, ssa_51, ssa_38
	vec1 32 ssa_57 = b32csel ssa_54, ssa_52, ssa_39
	vec1 32 ssa_58 = b32csel ssa_54, ssa_53, ssa_40
	vec4 32 ssa_59 = vec4 ssa_55, ssa_56, ssa_57, ssa_58
	intrinsic store_output (ssa_59, ssa_6) (8, 15, 0, 160, 132) /* base=8 */ /* wrmask=xyzw */ /* component=0 */ /* src_type=float32 */ /* location=4 slots=1 */	/* color */
	/* succs: block_1 */
	block block_1:
}
```
All the `b32csel` ops are generated by the above NIR pass, with each one "checking" a non-constant index against a constant value. At the end of the shader, the `store_output` uses the correct values, but this is pretty gross.

## And Then Inlining
Some time ago, noted Gallium professor Marek Olšák authored a [series](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/6946) which provided a codepath for inlining uniform data directly into shaders. The process for this is two steps:
* Detect and designate uniforms to be inlined
* Replace shader loads of these uniforms with the actual uniforms

The purpose of this is specifically to eliminate complex conditionals resulting from uniform data, so the detection NIR pass specifically looks for conditionals which use only constants and uniform data as the sources. Something like `if (uniform_variable_expression)` then becomes `if (constant_value_expression)` which can then be optimized out, greatly simplifying the eventual shader instructions.

Looking at the above NIR, this seems like a good target for inlining as well, so I took my hatchet to the detection pass and added in support for the `bcsel` and `fcsel` ALU ops. The results are good, to say the least:

```
shader: MESA_SHADER_FRAGMENT
inputs: 0
outputs: 0
uniforms: 8
ubos: 5
shared: 0
decl_var shader_out INTERP_MODE_NONE vec4 color (FRAG_RESULT_DATA0.xyzw, 8, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec1 32 ssa_0 = load_const (0x00000001 /* 0.000000 */)
	vec1 32 ssa_1 = load_const (0x00000004 /* 0.000000 */)
	vec1 32 ssa_2 = load_const (0x00000010 /* 0.000000 */)
	vec1 32 ssa_3 = intrinsic load_ubo (ssa_0, ssa_2) (0, 1073741824, 16, 0, -1) /* access=0 */ /* align_mul=1073741824 */ /* align_offset=16 */ /* range_base=0 */ /* range=-1 */
	vec1 32 ssa_4 = ishl ssa_3, ssa_1
	vec1 32 ssa_5 = load_const (0x00000002 /* 0.000000 */)
	vec1 32 ssa_6 = load_const (0xfffffffc /* -nan */)
	vec1 32 ssa_7 = iand ssa_4, ssa_6
	vec1 32 ssa_8 = load_const (0x00000000 /* 0.000000 */)
	vec4 32 ssa_9 = intrinsic load_ubo (ssa_5, ssa_7) (0, 4, 0, 0, -1) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */ /* range_base=0 */ /* range=-1 */
	intrinsic store_output (ssa_9, ssa_8) (8, 15, 0, 160, 132) /* base=8 */ /* wrmask=xyzw */ /* component=0 */ /* src_type=float32 */ /* location=4 slots=1 */	/* color */
	/* succs: block_1 */
	block block_1:
}
```
The second `load_ubo` here is using the inlined uniform data to determine that it needs to load the `0` index, greatly reducing the shader's complexity.