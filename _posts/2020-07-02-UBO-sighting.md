---
published: true
title: 'UBO Sighting'
---
## At last
It's time to finish up UBO support in this long form patch review blog series. Here's where the current progress has left things:

```c

static void
emit_load_ubo(struct ntv_context *ctx, nir_intrinsic_instr *intr)
{
   nir_const_value *const_block_index = nir_src_as_const_value(intr->src[0]);
   assert(const_block_index); // no dynamic indexing for now

   nir_const_value *const_offset = nir_src_as_const_value(intr->src[1]);
   if (const_offset) {
      SpvId uvec4_type = get_uvec_type(ctx, 32, 4);
      SpvId pointer_type = spirv_builder_type_pointer(&ctx->builder,
                                                      SpvStorageClassUniform,
                                                      uvec4_type);

      unsigned idx = const_offset->u32 / 16;
      SpvId member = emit_uint_const(ctx, 32, 0);
      SpvId offset = emit_uint_const(ctx, 32, idx);
      SpvId offsets[] = { member, offset };
      SpvId ptr = spirv_builder_emit_access_chain(&ctx->builder, pointer_type,
                                                  ctx->ubos[const_block_index->u32], offsets,
                                                  ARRAY_SIZE(offsets));
      SpvId result = spirv_builder_emit_load(&ctx->builder, uvec4_type, ptr);

      SpvId type = get_dest_uvec_type(ctx, &intr->dest);
      unsigned num_components = nir_dest_num_components(intr->dest);
      if (num_components == 1) {
         uint32_t components[] = { 0 };
         result = spirv_builder_emit_composite_extract(&ctx->builder,
                                                       type,
                                                       result, components,
                                                       1);
      } else if (num_components < 4) {
         SpvId constituents[num_components];
         SpvId uint_type = spirv_builder_type_uint(&ctx->builder, 32);
         for (uint32_t i = 0; i < num_components; ++i)
            constituents[i] = spirv_builder_emit_composite_extract(&ctx->builder,
                                                                   uint_type,
                                                                   result, &i,
                                                                   1);

         result = spirv_builder_emit_composite_construct(&ctx->builder,
                                                         type,
                                                         constituents,
                                                         num_components);
      }

      if (nir_dest_bit_size(intr->dest) == 1)
         result = uvec_to_bvec(ctx, result, num_components);

      store_dest(ctx, &intr->dest, result, nir_type_uint);
   } else
      unreachable("uniform-addressing not yet supported");
}
```
Remaining work here:
* handle dynamic offsets in order to e.g., process shaders which use loops to access a UBO member index
* handle loading an index inside each `vec4`-sized UBO member in order to be capable of accessing components

## More problems
There's another tangle here when it comes to accessing components of a UBO member though. The `Extract` operations in `SPIR-V` all take literals, not `SPIR-V` ids, which means they can't be used to support dynamic offsets from shader-side variables. As a result, [OpAccessChain](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#OpAccessChain) is the best solution, but this has some small challenges itself.

The way that `OpAccessChain` works is that it takes an array of index values that are used to progressively access deeper parts of a composite type. For a case like a `vec4[2]`, passing `[0, 2]` as the index array would access the first vec4's third member, as this Op delves based on the composite's type.

However, in `emit_load_ubo()` here, the instructions passed provide the offset in bytes, not "members". This means the value passed as `src[1]` here has to be converted from bytes into "members", and it has to be done such that `OpAccessChain` gets three separate index values in order to access the exact component of the UBO that the instruction specifies. The calculation is familiar for people who have worked extensively in C:
```
index_0 = 0;
index_1 = offset / sizeof(vec4)
index_2 = (offset % sizeof(vec4) / sizeof(int)
```
* The first index is always 0 since the type is a pointer.
* The second index is determining which `vec4` to access; since all UBO members are sized as `vec4` types, this is effectively determining which member of the UBO to access
* The third index accesses components of the variable, which in `SPIR-V` internals has been sized by `ntv` as an int-based type in terms of size

This is it for loading the UBO, but now those loaded values need to be stored, as that's the point of this instruction. In the above code segment, the entire `vec4` is loaded, and then [OpCompositeExtract](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#OpCompositeExtract) is used to grab the desired values, creating a new composite at the end for storage. This won't work for dynamic usage, however, as I mentioned previously: `OpCompositeExtract` takes literal index values, which means it can only be used with constant offsets.

Instead, a solution which handles both cases would be to use the `OpAccessChain` to loop over each individual component that needs to be loaded, Then these loaded values can be reassembled into a composite at the end.

## More code
The end result looks like this:
```c
static void
emit_load_ubo(struct ntv_context *ctx, nir_intrinsic_instr *intr)
{
   nir_const_value *const_block_index = nir_src_as_const_value(intr->src[0]);
   assert(const_block_index); // no dynamic indexing for now

   SpvId uint_type = get_uvec_type(ctx, 32, 1);
   SpvId one = emit_uint_const(ctx, 32, 1);

   /* number of components being loaded */
   unsigned num_components = nir_dest_num_components(intr->dest);
   SpvId constituents[num_components];
   SpvId result;

   /* destination type for the load */
   SpvId type = get_dest_uvec_type(ctx, &intr->dest);
   /* an id of the array stride in bytes */
   SpvId vec4_size = emit_uint_const(ctx, 32, sizeof(uint32_t) * 4);
   /* an id of an array member in bytes */
   SpvId uint_size = emit_uint_const(ctx, 32, sizeof(uint32_t));

   /* we grab a single array member at a time, so it's a pointer to a uint */
   SpvId pointer_type = spirv_builder_type_pointer(&ctx->builder,
                                                   SpvStorageClassUniform,
                                                   uint_type);

   /* our generated uniform has a memory layout like
    *
    * struct {
    *    vec4 base[array_size];
    * };
    *
    * where 'array_size' is set as though every member of the ubo takes up a vec4,
    * even if it's only a vec2 or a float.
    *
    * first, access 'base'
    */
   SpvId member = emit_uint_const(ctx, 32, 0);
   /* this is the offset (in bytes) that we're accessing:
    * it may be a const value or it may be dynamic in the shader
    */
   SpvId offset = get_src(ctx, &intr->src[1]);
   /* convert offset to an array index for 'base' to determine which vec4 to access */
   SpvId vec_offset = emit_binop(ctx, SpvOpUDiv, uint_type, offset, vec4_size);
   /* use the remainder to calculate the byte offset in the vec, which tells us the member
    * that we're going to access
    */
   SpvId vec_member_offset = emit_binop(ctx, SpvOpUDiv, uint_type,
                                        emit_binop(ctx, SpvOpUMod, uint_type, offset, vec4_size),
                                        uint_size);
   /* OpAccessChain takes an array of indices that drill into a hierarchy based on the type:
    * index 0 is accessing 'base'
    * index 1 is accessing 'base[index 1]'
    * index 2 is accessing 'base[index 1][index 2]'
    *
    * we must perform the access this way in case src[1] is dynamic because there's
    * no other spirv method for using an id to access a member of a composite, as
    * (composite|vector)_extract both take literals
    */
   for (unsigned i = 0; i < num_components; i++) {
      SpvId indices[3] = { member, vec_offset, vec_member_offset };
      SpvId ptr = spirv_builder_emit_access_chain(&ctx->builder, pointer_type,
                                                  ctx->ubos[const_block_index->u32], indices,
                                                  ARRAY_SIZE(indices));
      /* load a single value into the constituents array */
      constituents[i] = spirv_builder_emit_load(&ctx->builder, uint_type, ptr);
      /* increment to the next vec4 member index for the next load */
      vec_member_offset = emit_binop(ctx, SpvOpIAdd, uint_type, vec_member_offset, one);
   }

   /* if loading more than 1 value, reassemble the results into the desired type,
    * otherwise just use the loaded result
    */
   if (num_components > 1) {
      result = spirv_builder_emit_composite_construct(&ctx->builder,
                                                      type,
                                                      constituents,
                                                      num_components);
   } else
      result = constituents[0];

   /* explicitly convert to a bool vector if the destination type is a bool */
   if (nir_dest_bit_size(intr->dest) == 1)
      result = uvec_to_bvec(ctx, result, num_components);

   store_dest(ctx, &intr->dest, result, nir_type_uint);
}
```
But wait! Perhaps some avid reader is now considering how many load operations are potentially being added by this method if the original instruction was intended to load an entire `vec4`. Surely some optimizing can be done here?

One of the great parts about `ntv` is that there's not much need to optimize anything in advance here. Getting things working is usually "good enough", and the reason for that is once again `NIR`. While it's true that loading a `vec4` member of a UBO from this code does generate four `load_ubo` instructions, these instructions will get automatically optimized back to a single `load_ubo` by a `nir_lower_io` pass triggered from the underlying Vulkan driver, which means spending any effort pre-optimizing here is wasted time.

## Moving on
ARB_uniform_buffer_object is done now, so look forward to new topics again.
