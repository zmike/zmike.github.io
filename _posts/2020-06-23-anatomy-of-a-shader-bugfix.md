---
published: false
---
## Something different

I've talked and rambled about various things, and maybe I've given an idea of what it's like to work on Zink, but the reality is that I spend a majority of my time working on the shader translation pipeline, which converts NIR to SPIRV.

Today let's go through the process for fixing an issue in this pipeline, as detected by piglit.

## Steps
* build piglit, as described by its [README](https://gitlab.freedesktop.org/mesa/piglit)
* run piglit; my current run is executed with `VK_INSTANCE_LAYERS= MESA_GLSL_CACHE_DISABLE=true MESA_LOADER_DRIVER_OVERRIDE=zink ./piglit run --timeout 3000 gpu results/new`
* generate a viewable summary with e.g., `./piglit summary html results/compare <possibly some previous results> results/new`
* open generated `results/compare/index.html` in browser

Now there's a massive list of tests with pass/fail results. Clicking on the results of any test will provide more detail, just like this:

![shot-2020-06-23_15-35-31.png]({{site.baseurl}}/_posts/shot-2020-06-23_15-35-31.png)

In this case, `spec@glsl-1.30@execution@fs-texelfetchoffset-2d` is failing. What does that mean?

## Debugging piglit tests
Near the bottom of the above results, there's a row for **Command**, which is the command used to run a given test. This command can be run in any tool, such as `gdb` or `valgrind`, in order to run only this test.

More importantly, however, in the case of shader tests, it lets someone debugging a given test produce NIR output, as this is usually the best way to figure out what's going wrong.

To do so for the above test, I've run `NIR_PRINT=1 LIBGL_DEBUG=verbose MESA_GLSL_CACHE_DISABLE=true MESA_LOADER_DRIVER_OVERRIDE=zink bin/fs-texelFetchOffset-2D -auto -fbo &>| zink`. This generates a file `zink` in my current directory which contains the generated NIR as it progreses through various stages of optimization and translation.

## NIR
The specified test is for a fragment shader, as indicated by `fs` in the name or just reading the test code, which uses the following shader:
```glsl
"#version 130\n"
"uniform ivec2 pos;\n"
"uniform int lod;\n"
"uniform sampler2D tex;\n"
"void main()\n"
"{\n"
"       const ivec2 offset = ivec2(-2, 2);\n"
"       vec4 texel = texelFetchOffset(tex, pos, lod, offset);\n"
"	gl_FragColor = texel;\n"
"}\n";
```

Searching through the NIR output for the last output of the fragment shader IR yields:

```
shader: MESA_SHADER_FRAGMENT
inputs: 0
outputs: 0
uniforms: 8
ubos: 1
shared: 0
decl_var uniform INTERP_MODE_NONE sampler2D tex (~0, 0, 672)
decl_var ubo INTERP_MODE_NONE struct_uniform_0 uniform_0 (~0, 0, 640)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[0] (FRAG_RESULT_DATA0.xyzw, 8, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec1 32 ssa_0 = load_const (0x00000002 /* 0.000000 */)
	vec1 32 ssa_1 = load_const (0x00000000 /* 0.000000 */)
	vec4 32 ssa_2 = intrinsic load_ubo (ssa_0, ssa_1) (0, 4, 0) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */
	vec1 32 ssa_3 = load_const (0x00000010 /* 0.000000 */)
	vec4 32 ssa_4 = intrinsic load_ubo (ssa_0, ssa_3) (0, 16, 0) /* access=0 */ /* align_mul=16 */ /* align_offset=0 */
	vec2 32 ssa_5 = vec2 ssa_2.x, ssa_2.y
	vec1 32 ssa_6 = mov ssa_4.x
	vec4 32 ssa_7 = (float)txf ssa_5 (coord), ssa_6 (lod), 3 (texture), 0 (sampler)
	intrinsic store_output (ssa_7, ssa_1) (8, 15, 0, 160) /* base=8 */ /* wrmask=xyzw */ /* component=0 */ /* type=float32 */	/* gl_FragData[0] */
	/* succs: block_1 */
	block block_1:
}
```

That's a lot to go through in a single post, so I'll be providing a brief overview for now. The most important thing to keep in mind is that `ssa_*` values in IR are [SSA](https://en.wikipedia.org/wiki/Static_single_assignment_form), so each value can be traced through execution by following the assignments.

Looking at `main` in the shader code, an `ivec2` is created as (-2, 2), and this is passed into `texelFetchOffset()` as the offset from the `pos` uniform.

Looking at `main` in the IR, the first 5 lines of `block_0` (the only block) are used to load resources. It can be assumed they're generally correct right now, though that won't always be the case. Next there's a vec2 formed (`ssa_5`) from the `load_ubo`-created ssa_2; as can be seen a couple lines down, this is is the `coord` or *P* param in [texelFetchOffset](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/texelFetchOffset.xhtml), which is abbreviated as `txf` here.

In particular, `ssa_5` is formed and then passed directly to the `txf` instruction. What happened to the offset?

Let's check out NIR generated for this shader by IRIS, the Intel gallium driver:
```
shader: MESA_SHADER_FRAGMENT
name: GLSL3
inputs: 0
outputs: 1
uniforms: 0
ubos: 1
shared: 0
decl_var uniform INTERP_MODE_NONE ivec2 pos (1, 0, 0)
decl_var uniform INTERP_MODE_NONE int lod (2, 2, 0)
decl_var uniform INTERP_MODE_NONE sampler2D tex (3, 0, 0)
decl_var ubo INTERP_MODE_NONE vec4[1] uniform_0 (0, 0, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragColor (FRAG_RESULT_COLOR.xyzw, 4, 0)
decl_function main (0 params)

impl main {
	block block_0:
	/* preds: */
	vec1 32 ssa_0 = load_const (0x00000000 /* 0.000000 */)
	vec1 32 ssa_1 = load_const (0x00000002 /* 0.000000 */)
	vec3 32 ssa_2 = intrinsic load_ubo (ssa_1, ssa_0) (0, 4, 0) /* access=0 */ /* align_mul=4 */ /* align_offset=0 */
	vec1 32 ssa_16 = mov ssa_2.z
	vec1 32 ssa_3 = load_const (0xfffffffe /* -nan */)
	vec1 32 ssa_6 = iadd ssa_2.x, ssa_3
	vec1 32 ssa_7 = iadd ssa_2.y, ssa_1
	vec2 32 ssa_8 = vec2 ssa_6, ssa_7
	vec4 32 ssa_9 = (float)txf ssa_8 (coord), ssa_16 (lod), 1 (texture), 0 (sampler)
	intrinsic store_output (ssa_9, ssa_0) (4, 15, 0, 160) /* base=4 */ /* wrmask=xyzw */ /* component=0 */ /* type=float32 */	/* gl_FragColor */
	/* succs: block_1 */
	block block_1:
}
```
In particular:
```
	vec1 32 ssa_6 = iadd ssa_2.x, ssa_3
	vec1 32 ssa_7 = iadd ssa_2.y, ssa_1
	vec2 32 ssa_8 = vec2 ssa_6, ssa_7
```
As can be seen here, the ssa for the `coord` param is only formed after a pair of `iadd` instructions occur, as one would expect to see if a vec2 offset were added to a vec2 coordinate.

Indeed, it seems that Zink is ignoring the offset here.

## Fixing
Armed with the knowledge that a `txf` instruction is involved, a quick search through `nir_to_spirv.c` reveals the `emit_tex()` function as a likely starting point, as it's where `txf` is handled.