---
published: true
---
## Shader ALUs

Today let's talk about ALUs a little.

During shader compilation, `GLSL` gets serialized into `SSA` form, which is what `ntv` operates on when translating it into `SPIR-V`. An ALU in the context of Zink (specifically `ntv`) is an algebraic operation which takes a varying number of inputs and generates an output. This is represented in `NIR` by a struct, `nir_alu_instr`, which contains the operation type, the inputs, and the output.

When writing GLSL, there's the general assumption that writing something like `1 + 2` will yield `3`, but this is contingent on the driver being able to correctly compile the `NIR` form of the shader into instructions that the physical hardware runs in order to get that result. In Zink, there's the need to translate all these `NIR` instructions into `SPIR-V`, which is sometimes made trickier by both different semantics between similar `GLSL` and `SPIR-V` operations as well as aggressive `NIR` optimizations.

## A deep dive into isnan()
The [isnan](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/isnan.xhtml) function checks whether the input is a number. It's a simple enough functionality to describe, but the implementation and transit through the GLSL->NIR->SPIR-V->NIR pipeline is fraught with perils.

In mesa, `isnan(x)` is serialized to `NIR` as `fne(x, x)`, where `fne` is the operation for float-not-equal, which compares two floats to determine whether they are equal. As such, there's never actually a case where `isnan` gets passed through `ntv`. Let's see what this looks like in practice with this failing shader test:

```
// from piglit's fs-isnan-vec2.shader_test for GLSL 1.30
#version 130
uniform vec2 numerator;
uniform vec2 denominator;

void main()
{
  gl_FragColor = vec4(isnan(numerator/denominator), 0.0, 1.0);
}
```
In Zink, this yields:
```
shader: MESA_SHADER_FRAGMENT
inputs: 0
outputs: 0
uniforms: 0
ubos: 1
shared: 0
decl_var ubo INTERP_MODE_NONE struct_uniform_0 uniform_0 (~0, 0, 640)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragColor (FRAG_RESULT_COLOR.xyzw, 4, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec4 32 ssa_1 = load_const (0x00000000 /* 0.000000 */, 0x00000000 /* 0.000000 */, 0x00000000 /* 0.000000 */, 0x3f800000 /* 1.000000 */)
	vec1 32 ssa_2 = load_const (0x00000000 /* 0.000000 */)
	intrinsic store_output (ssa_1, ssa_2) (8, 15, 0, 160) /* base=8 */ /* wrmask=xyzw */ /* component=0 */ /* type=float32 */	/* gl_FragColor */
	/* succs: block_1 */
	block block_1:
}
```
As with yesterday's shader adventure, here's IRIS as a control:
```
shader: MESA_SHADER_FRAGMENT
name: GLSL3
inputs: 0
outputs: 1
uniforms: 0
ubos: 1
shared: 0
decl_var uniform INTERP_MODE_NONE vec2 numerator (0, 0, 0)
decl_var uniform INTERP_MODE_NONE vec2 denominator (1, 2, 0)
decl_var ubo INTERP_MODE_NONE vec4[1] uniform_0 (0, 0, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragColor (FRAG_RESULT_COLOR.xyzw, 4, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec1 32 ssa_0 = load_const (0x00000000 /* 0.000000 */)
	vec1 32 ssa_1 = load_const (0x3f800000 /* 1.000000 */)
	vec1 32 ssa_2 = load_const (0x00000001 /* 0.000000 */)
	vec4 32 ssa_3 = intrinsic load_ubo (ssa_2, ssa_0) (0, 4, 0) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */
	vec1 32 ssa_6 = frcp ssa_3.z
	vec1 32 ssa_7 = frcp ssa_3.w
	vec1 32 ssa_8 = fmul ssa_3.x, ssa_6
	vec1 32 ssa_9 = fmul ssa_3.y, ssa_7
	vec1 32 ssa_10 = fne32 ssa_8, ssa_8
	vec1 32 ssa_12 = b2f32 ssa_10
	vec1 32 ssa_11 = fne32 ssa_9, ssa_9
	vec1 32 ssa_13 = b2f32 ssa_11
	vec4 32 ssa_14 = vec4 ssa_12, ssa_13, ssa_0, ssa_1
	intrinsic store_output (ssa_14, ssa_0) (4, 15, 0, 160) /* base=4 */ /* wrmask=xyzw */ /* component=0 */ /* type=float32 */	/* gl_FragColor */
	/* succs: block_1 */
	block block_1:
}
```

This is clearly much different. In particular, note that IRIS retains its `fne` instructions, but Zink has lost them along the way.

## Why is this?
The problem comes from how `SPIR-V` is translated back to `NIR`. When emitting `fne(a, a)` into `SPIR-V` with [OpFOrdNotEqual](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#OpFOrdNotEqual), the result is that the NaN-ness is ignored, and the NaN value is compared against itself, managing to be equivalent somehow, which breaks the test. This is due to how `OpFOrdNotEqual` is explicitly used for **ordered** (numeric).

Using [OpFUnordNotEqual](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#OpFUnordNotEqual) for this case has no such issue, as this op always return false if either of the inputs are unordered (NaN).
