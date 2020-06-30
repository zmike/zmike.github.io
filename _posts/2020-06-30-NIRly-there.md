---
published: false
---
## Getting right to it

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
This is the top half of `emit_load_ubo()`, which performs the load on the desired memory region for the UBO access.