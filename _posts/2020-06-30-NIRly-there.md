---
published: true
---
## assert()gery

In yesterday's post, I left off in saying that removing an `assert()` from the constant block index check wasn't going to work quite right. Let's see why that is.

Some context again:
```c
static void
emit_load_ubo(struct ntv_context *ctx, nir_intrinsic_instr *intr)
{
   nir_const_value *const_block_index = nir_src_as_const_value(intr->src[0]);
   assert(const_block_index); // no dynamic indexing for now
   assert(const_block_index->u32 == 0); // we only support the default UBO for now

   nir_const_value *const_offset = nir_src_as_const_value(intr->src[1]);
   if (const_offset) {
      SpvId uvec4_type = get_uvec_type(ctx, 32, 4);
      SpvId pointer_type = spirv_builder_type_pointer(&ctx->builder,
                                                      SpvStorageClassUniform,
                                                      uvec4_type);

      unsigned idx = const_offset->u32;
      SpvId member = emit_uint_const(ctx, 32, 0);
      SpvId offset = emit_uint_const(ctx, 32, idx);
      SpvId offsets[] = { member, offset };
      SpvId ptr = spirv_builder_emit_access_chain(&ctx->builder, pointer_type,
                                                  ctx->ubos[0], offsets,
                                                  ARRAY_SIZE(offsets));
      SpvId result = spirv_builder_emit_load(&ctx->builder, uvec4_type, ptr);
```
This is the top half of `emit_load_ubo()`, which performs the load on the desired memory region for the UBO access. In particular, the line I'm going to be exploring today is
```c
   assert(const_block_index->u32 == 0); // we only support the default UBO for now
```
Which directly corresponds to the explicit `0` in
```c
      SpvId ptr = spirv_builder_emit_access_chain(&ctx->builder, pointer_type,
                                                  ctx->ubos[0], offsets,
                                                  ARRAY_SIZE(offsets));
```
At a glance, it seems like the `assert()` can be removed, and `const_block_index->u32` can be passed as the index to the `ctx->ubos` array, which is where all the declared UBO variables are stored, and there won't be any issue.

Not so.

In fact, there's a number of problems with this.

## NIR resurfaces
Over in `zink_compiler.c`, Zink runs a `nir_lower_uniforms_to_ubo` pass on shaders. What this pass does is:
* rewrites all `load_uniform` instructions as `load_ubo` instructions for the UBO bound to 0, which works with Gallium's merging of all non-block uniforms into UBO with binding point 0 (which is what's currently handled by Zink)
* adds the variable for a UBO with binding point 0 if there's any `load_uniform` instructions
* increments the binding points (and load instructions) of all existing UBOs by 1
* uses a specified multiplier to rewrite the offset values specified by the converted `load_ubo` instructions which were previously `load_uniform`

But then there's a problem: what happens when this pass gets run when there's no non-block uniforms? Well, the answer is just as expected:
* ~~rewrites all `load_uniform` instructions as `load_ubo` instructions for the UBO bound to 0, which works with Gallium's merging of all non-block uniforms into UBO with binding point 0 (which is what's currently handled by Zink)~~
* ~~adds the variable for a UBO with binding point 0 if there's any `load_uniform` instructions~~
* increments the binding points (and load instructions) of all existing UBOs by 1
* ~~uses a specified multiplier to rewrite the offset values specified by the converted `load_ubo` instructions which were previously `load_uniform`~~

So now, in `emit_load_ubo()` above, that `ctx->ubos[const_block_index->u32]` is actually going to translate to `ctx->ubos[1]` in the case of a shader without any uniforms. Unfortunately, here's the function which declares the UBO variables:

```c
static void
emit_ubo(struct ntv_context *ctx, struct nir_variable *var)
{
   uint32_t size = glsl_count_attribute_slots(var->type, false);
   SpvId vec4_type = get_uvec_type(ctx, 32, 4);
   SpvId array_length = emit_uint_const(ctx, 32, size);
   SpvId array_type = spirv_builder_type_array(&ctx->builder, vec4_type,
                                               array_length);
   spirv_builder_emit_array_stride(&ctx->builder, array_type, 16);

   // wrap UBO-array in a struct
   SpvId struct_type = spirv_builder_type_struct(&ctx->builder, &array_type, 1);
   if (var->name) {
      char struct_name[100];
      snprintf(struct_name, sizeof(struct_name), "struct_%s", var->name);
      spirv_builder_emit_name(&ctx->builder, struct_type, struct_name);
   }

   spirv_builder_emit_decoration(&ctx->builder, struct_type,
                                 SpvDecorationBlock);
   spirv_builder_emit_member_offset(&ctx->builder, struct_type, 0, 0);


   SpvId pointer_type = spirv_builder_type_pointer(&ctx->builder,
                                                   SpvStorageClassUniform,
                                                   struct_type);

   SpvId var_id = spirv_builder_emit_var(&ctx->builder, pointer_type,
                                         SpvStorageClassUniform);
   if (var->name)
      spirv_builder_emit_name(&ctx->builder, var_id, var->name);

   assert(ctx->num_ubos < ARRAY_SIZE(ctx->ubos));
   ctx->ubos[ctx->num_ubos++] = var_id;

   spirv_builder_emit_descriptor_set(&ctx->builder, var_id,
                                     var->data.descriptor_set);
   int binding = zink_binding(ctx->stage,
                              VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
                              var->data.binding);
   spirv_builder_emit_binding(&ctx->builder, var_id, binding);
}
```
Specifically:
```c
   ctx->ubos[ctx->num_ubos++] = var_id;
```
Indeed, this is zero-indexed, which means all the UBO access for a shader with no uniforms is going to fail because all the UBO load instructions are using a block index that's off by one.

## Solved
As is my way, I slapped some `if (!shader->num_uniforms)` flex tape on running the zink_compiler `nir_lower_uniforms_to_ubo` pass in order to avoid potentially breaking the pass's other usage over in TGSI by changing the pass itself, and now the problem is solved. The `assert()` can now be removed.

Yes, sometimes there's all this work, and analyzing, and debugging, and blogging, and the end result is a sweet, sweet zero/null check.

Tune in next time when I again embark on a journey that definitely, in no way, results in more flex tape being used.
