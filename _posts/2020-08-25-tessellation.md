---
published: false
---
## The Tessellating Commences

[Tessellation shaders](https://www.khronos.org/opengl/wiki/Tessellation) were the second type of new shader I worked to implement after geometry shaders, which I haven't blogged about yet.

As I always say, why start at the beginning when I could start at the middle, then go back to the beginning, then skip right to the end.

Tessellation happens across two distinct shader stages: tessellation control (TCS) and tessellation evaluation (TES). In short, the former sets up parameters for the latter to generate vertex data with. In OpenGL, TCS can be omitted by specifying default inner and outer levels with [glPatchParameterfv](https://www.khronos.org/opengl/wiki/Tessellation#Tessellation_levels) which will then be propagated to `gl_TessLevelInner` and `gl_TessLevelOuter`, respectively.

Vulkan, however, will only perform tessellation when both TCS and TES are present for a given graphics pipeline.

There are piglit tests which verify GL's TCS-less tessellation capability.

Uh-oh.

## Faking It
As is the case with many things in this project, Vulkan's lack of support for the "easier" OpenGL method of doing things requires jumping through a number of hoops. In this case, the hoops amount to **getting default inner/outer levels from gallium states into the TES stage**.

Going into this problem, I knew nothing about any of these functionalities or shaders (or even that they existed), but I quickly learned a number of key points:
* TES implicitly expects to read `gl_TessLevelInner` and `gl_TessLevelOuter` variables
* TCS expands and propagates all VS outputs to something like `type output_var[gl_MaxPatchVertices]` for the TES to read
* Vulkan has no concept of "default" levels
* Push constants exist

The last of these was the key to my approach. In short, if a TCS doesn't exist, I just need to make one. Since I already have full control of the pipeline state, I know when a TCS is absent, and I also know the default tessellation levels. Thus, I can emit them to a shader using a push constant, and I can also create my own passthrough TCS stage to jam into the pipeline.

Ideally, this passthrough shader would look a bit like this pseudoglsl:
```glsl
#version 150
#extension GL_ARB_tessellation_shader : require

in vec4 some_var[gl_MaxPatchVertices];
out vec4 some_var_out;

layout(push_constant) uniform tcsPushConstants {
    layout(offset = 0) float TessLevelInner[2];
    layout(offset = 8) float TessLevelOuter[4];
} u_tcsPushConstants;
layout(vertices = $vertices_per_patch) out;
void main()
{
  gl_TessLevelInner = u_tcsPushConstants.TessLevelInner;
  gl_TessLevelOuter = u_tcsPushConstants.TessLevelOuter;
  some_var_out = some_var[gl_InvocationID];
}
```
where all input variables get expanded to `gl_MaxPatchVertices`, and the default levels are pulled from the push constant block.

## Shader Construction
Now, it's worthwhile to note that at this point, `ntv` had no ability to load push constants. With that said, I've alraedy blogged extensively about the process for UBO handling, and the mechanisms for both types of load are more or less identical. This means that as long as I structure my push constant data in the same way that I've expected my UBOs to look (i.e., a struct containing an array of vec4s), I can just copy and paste the existing code with a couple minor tweaks and simplifications since I know this codepath will only be hit by fixed function shader code that I've personally verified.

Therefore, I don't actually have to blog about the specific loading mechanism, and I can just refer [to that post]({{site.url}}/UBO-sighting) and skip to the juicy stuff.

Specifically, I'm talking about writing the above tessellation control shader in NIR.

As a quick note, I've created (in patches after the initial implementation) a struct for managing the overall push constant data, which at this point would look like this:
```c
struct zink_push_constant {
   float default_inner_level[2];
   float default_outer_level[4];
};
```

Now, let's jump in.
```c
nir_shader *nir = nir_shader_create(NULL, MESA_SHADER_TESS_CTRL, &nir_options, NULL);
nir_function *fn = nir_function_create(nir, "main");
fn->is_entrypoint = true;
nir_function_impl *impl = nir_function_impl_create(fn);
```
First, I've created a new shader for the TCS stage, and I've added a "main" function that's flagged as a shader entrypoint. This is the base of every shader.
```c
nir_builder b;
nir_builder_init(&b, impl);
b.cursor = nir_before_block(nir_start_block(impl));
```
Here, a `nir_builder` is instantiated for the shader. `nir_builder` is the tool which allows modifying existing shaders. It operates using a "cursor" position, which determines where new instructions are being added.

To begin inserting instructions, the cursor is positioned at the start of the function block that's just been created.
```c
nir_intrinsic_instr *invocation_id = nir_intrinsic_instr_create(b.shader, nir_intrinsic_load_invocation_id);
nir_ssa_dest_init(&invocation_id->instr, &invocation_id->dest, 1, 32, "gl_InvocationID");
nir_builder_instr_insert(&b, &invocation_id->instr);
```
The first instruction is loading the `gl_InvocationID` variable, which is a single 32-bit integer. The instruction is created, then the destination is initialized to the type being stored since this instruction type loads a value, and then the instruction is inserted into the shader using the `nir_builder`.
```c
nir_foreach_shader_out_variable(var, vs->nir) {
   const struct glsl_type *type = var->type;
   const struct glsl_type *in_type = var->type;
   const struct glsl_type *out_type = var->type;
   char buf[1024];
   snprintf(buf, sizeof(buf), "%s_out", var->name);
   in_type = glsl_array_type(type, 32 /* MAX_PATCH_VERTICES */, 0);
   out_type = glsl_array_type(type, vertices_per_patch, 0);

   nir_variable *in = nir_variable_create(nir, nir_var_shader_in, in_type, var->name);
   nir_variable *out = nir_variable_create(nir, nir_var_shader_out, out_type, buf);
   out->data.location = in->data.location = var->data.location;
   out->data.location_frac = in->data.location_frac = var->data.location_frac;

   /* gl_in[] receives values from equivalent built-in output
      variables written by the vertex shader (section 2.14.7).  Each array
      element of gl_in[] is a structure holding values for a specific vertex of
      the input patch.  The length of gl_in[] is equal to the
      implementation-dependent maximum patch size (gl_MaxPatchVertices).
      - ARB_tessellation_shader
    */
   for (unsigned i = 0; i < vertices_per_patch; i++) {
      /* we need to load the invocation-specific value of the vertex output and then store it to the per-patch output */
      nir_if *start_block = nir_push_if(&b, nir_ieq(&b, &invocation_id->dest.ssa, nir_imm_int(&b, i)));
      nir_deref_instr *in_array_var = nir_build_deref_array(&b, nir_build_deref_var(&b, in), &invocation_id->dest.ssa);
      nir_ssa_def *load = nir_load_deref(&b, in_array_var);
      nir_deref_instr *out_array_var = nir_build_deref_array_imm(&b, nir_build_deref_var(&b, out), i);
      nir_store_deref(&b, out_array_var, load, 0xff);
      nir_pop_if(&b, start_block);
   }
}
```
Next, all the output variables for the vertex shader are looped over and expanded to arrays of `MAX_PATCH_VERTICES` size, which in the case of mesa is 32. These values are loaded based on `gl_InvocationID`, so the stores for the expanded variable values are always conditional on the current invocation id value matching the [vertices_per_patch](https://www.khronos.org/opengl/wiki/Tessellation#Patches) index, as is checked in the `nir_push_if` line where a conditional is added for this.
```c
nir_variable *gl_TessLevelInner = nir_variable_create(nir, nir_var_shader_out, glsl_array_type(glsl_float_type(), 2, 0), "gl_TessLevelInner");
gl_TessLevelInner->data.location = VARYING_SLOT_TESS_LEVEL_INNER;
gl_TessLevelInner->data.patch = 1;
nir_variable *gl_TessLevelOuter = nir_variable_create(nir, nir_var_shader_out, glsl_array_type(glsl_float_type(), 4, 0), "gl_TessLevelOuter");
gl_TessLevelOuter->data.location = VARYING_SLOT_TESS_LEVEL_OUTER;
gl_TessLevelOuter->data.patch = 1;
```
Following this, the `gl_TessLevelInner` and `gl_TessLevelOuter` variables must be created so they can be processed in `ntv` and converted to the corresponding variables that SPIRV and the underlying vulkan driver expect to see. All that's necessary for this case is to set the types of the variables, the slot location, and then flag them as patch variables.
```c
/* hacks so we can size these right for now */
struct glsl_struct_field *fields = ralloc_size(nir, 2 * sizeof(struct glsl_struct_field));
fields[0].type = glsl_array_type(glsl_uint_type(), 2, 0);
fields[0].name = ralloc_asprintf(nir, "gl_TessLevelInner");
fields[0].offset = 0;
fields[1].type = glsl_array_type(glsl_uint_type(), 4, 0);
fields[1].name = ralloc_asprintf(nir, "gl_TessLevelOuter");
fields[1].offset = 8;
nir_variable *pushconst = nir_variable_create(nir, nir_var_shader_in,
                                              glsl_struct_type(fields, 2, "struct", false), "pushconst");
pushconst->data.location = VARYING_SLOT_VAR0;
```
Next up, creating a variable to represent the push constant for the TCS stage that's going to be injected. SPIRV is always explicit about variable sizes, so it's important that everything match up with the size and offset in the push constant that's being emitted. The inner level value will be first in the push constant layout, so this has offset 0, and the outer level will be second, so this has an offset of `sizeof(gl_TessLevelInner)`, which is known to be 8.
```c
nir_intrinsic_instr *load_inner = nir_intrinsic_instr_create(b.shader, nir_intrinsic_load_push_constant);
load_inner->src[0] = nir_src_for_ssa(nir_imm_int(&b, offsetof(struct zink_push_constant, default_inner_level)));
nir_intrinsic_set_base(load_inner, offsetof(struct zink_push_constant, default_inner_level));
nir_intrinsic_set_range(load_inner, 8);
load_inner->num_components = 2;
nir_ssa_dest_init(&load_inner->instr, &load_inner->dest, 2, 32, "TessLevelInner");
nir_builder_instr_insert(&b, &load_inner->instr);

nir_intrinsic_instr *load_outer = nir_intrinsic_instr_create(b.shader, nir_intrinsic_load_push_constant);
load_outer->src[0] = nir_src_for_ssa(nir_imm_int(&b, offsetof(struct zink_push_constant, default_outer_level)));
nir_intrinsic_set_base(load_outer, offsetof(struct zink_push_constant, default_outer_level));
nir_intrinsic_set_range(load_outer, 16);
load_outer->num_components = 4;
nir_ssa_dest_init(&load_outer->instr, &load_outer->dest, 4, 32, "TessLevelOuter");
nir_builder_instr_insert(&b, &load_outer->instr);

for (unsigned i = 0; i < 2; i++) {
   nir_deref_instr *store_idx = nir_build_deref_array_imm(&b, nir_build_deref_var(&b, gl_TessLevelInner), i);
   nir_store_deref(&b, store_idx, nir_channel(&b, &load_inner->dest.ssa, i), 0xff);
}
for (unsigned i = 0; i < 4; i++) {
   nir_deref_instr *store_idx = nir_build_deref_array_imm(&b, nir_build_deref_var(&b, gl_TessLevelOuter), i);
   nir_store_deref(&b, store_idx, nir_channel(&b, &load_outer->dest.ssa, i), 0xff);
}
```
Finally, the push constant data can be set to the created variables. Each variable needs two parts:
* loading the push constant data
* storing the loaded data to the variable

Each load of the push constant data requires a base and a range along with a single source value. The base and the source are both the offset of the value being loaded (referenced from `struct zink_push_constant`), and the range is the size of the value, which is constant based on the variable.

Using the same mechanic as the first intrinsic instruction that was created, each intrinsic is created and has its destination value initialized based on the size and number of components of the type being loaded (first load 2 values, then 4 values). A `src` is created based on the struct offset, which determines the offset at which data will be loaded, and the instruction is inserted using the `nir_builder`.

Following this, the loaded value is then dereferenced for each array member and stored component-wise into the actual `gl_TessLevel*` variable.
```c
nir->info.tess.tcs_vertices_out = vertices_per_patch;
nir_validate_shader(nir, "created");

pushconst->data.mode = nir_var_mem_push_const;
```
At last, the finishing touches. The `vertices_per_patch` value is set so that SPIRV can process the shader, and the shader is then validated by NIR to ensure that I didn't screw anything up.

And then the push constant gets its variable type swizzled to being a push constant so that `ntv` can special case it. This is [very illegal](https://www.youtube.com/watch?v=I7Tps0M-l64&hd=1), and so it has to happen at the point when absolutely no other NIR passes will be run on the shader or everything will explode.

But it's totally safe, I promise.

With all this done, the only remaining step is to set up the push constant [in the Vulkan pipeline](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#descriptorsets-push-constants), which is trivial by comparison.

The TCS is picked up automatically after being jammed into the existing shader stages, and everything works like it should.