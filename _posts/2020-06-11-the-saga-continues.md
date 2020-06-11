---
published: true
---
## More Fixes
Yesterday I covered the problems related to handling `gl_PointSize` during the `SPIR-V` conversion. But there were still more problems to overcome.

Packed outputs are a thing. This is the case when a variable is partially captured by `xfb`, e.g., in the case where only the x coordinate is selected from a `vec4` output, it will be packed into the buffer as a single float. The original code assumed that all captured output types would match the original type, which is clearly not the case in the previously-described scenario.

A lot went into handling this case. Let's jump into some of the code.

Critical terminology for this post:
* `SpvId` - an id in `SPIR-V` which represents some other value, as denoted by `<id>` in the `SPIR-V` spec

## Improved variable creation
The first step here was to figure out what the heck the resulting output type would be. Having remapped the `struct pipe_stream_output::register_index` values, it's now possible to check the original variable types and use that when creating the `xfb` output.
```c
/* return the intended xfb output vec type based on base type and vector size */
static SpvId
get_output_type(struct ntv_context *ctx, unsigned register_index, unsigned num_components)
{
   const struct glsl_type *out_type = ctx->so_output_gl_types[register_index];
   enum glsl_base_type base_type = glsl_get_base_type(out_type);
   if (base_type == GLSL_TYPE_ARRAY)
      base_type = glsl_get_base_type(glsl_without_array(out_type));

   switch (base_type) {
   case GLSL_TYPE_BOOL:
      return get_bvec_type(ctx, num_components);

   case GLSL_TYPE_FLOAT:
      return get_fvec_type(ctx, 32, num_components);

   case GLSL_TYPE_INT:
      return get_ivec_type(ctx, 32, num_components);

   case GLSL_TYPE_UINT:
      return get_uvec_type(ctx, 32, num_components);

   default:
      break;
   }
   unreachable("unknown type");
   return 0;
}

/* for streamout create new outputs, as streamout can be done on individual components,
   from complete outputs, so we just can't use the created packed outputs */
static void
emit_so_info(struct ntv_context *ctx, unsigned max_output_location,
             const struct pipe_stream_output_info *so_info, struct pipe_stream_output_info *local_so_info)
{
   for (unsigned i = 0; i < local_so_info->num_outputs; i++) {
      struct pipe_stream_output so_output = local_so_info->output[i];
      SpvId out_type = get_output_type(ctx, so_output.register_index, so_output.num_components);
      SpvId pointer_type = spirv_builder_type_pointer(&ctx->builder,
                                                      SpvStorageClassOutput,
                                                      out_type);
      SpvId var_id = spirv_builder_emit_var(&ctx->builder, pointer_type,
                                            SpvStorageClassOutput);
      char name[10];

      snprintf(name, 10, "xfb%d", i);
      spirv_builder_emit_name(&ctx->builder, var_id, name);
      spirv_builder_emit_offset(&ctx->builder, var_id, (so_output.dst_offset * 4));
      spirv_builder_emit_xfb_buffer(&ctx->builder, var_id, so_output.output_buffer);
      spirv_builder_emit_xfb_stride(&ctx->builder, var_id, so_info->stride[so_output.output_buffer] * 4);

      /* output location is incremented by VARYING_SLOT_VAR0 for non-builtins in vtn
       */
      uint32_t location = so_info->output[i].register_index;
      spirv_builder_emit_location(&ctx->builder, var_id, location);

      /* note: gl_ClipDistance[4] can the 0-indexed member of VARYING_SLOT_CLIP_DIST1 here,
       * so this is still the 0 component
       */
      if (so_output.start_component)
         spirv_builder_emit_component(&ctx->builder, var_id, so_output.start_component);

      uint32_t *key = ralloc_size(NULL, sizeof(uint32_t));
      *key = (uint32_t)so_output.register_index << 2 | so_output.start_component;
      _mesa_hash_table_insert(ctx->so_outputs, key, (void *)(intptr_t)var_id);

      assert(ctx->num_entry_ifaces < ARRAY_SIZE(ctx->entry_ifaces));
      ctx->entry_ifaces[ctx->num_entry_ifaces++] = var_id;
   }
}
```
Here's the slightly changed code for `emit_so_info()` along with a new helper function. Note that there's now `so_info` and `local_so_info` being passed here: the former is the gallium-produced streamout info, and the latter is the one that's been re-translated back to `enum gl_varying_slot` using the code from yesterday's post.

The remapped value is passed into the helper function, which retrieves the previously-stored `struct glsl_type` and then returns an `SpvId` for the necessary `xfb` output type, which is what's now used to create the variable.

# Improved variable value emission
Now that the variables are correctly created, it's important to ensure that the correct value is being emitted.
```c
static void
emit_so_outputs(struct ntv_context *ctx,
                const struct pipe_stream_output_info *so_info, struct pipe_stream_output_info *local_so_info)
{
   SpvId loaded_outputs[VARYING_SLOT_MAX] = {};
   for (unsigned i = 0; i < local_so_info->num_outputs; i++) {
      uint32_t components[NIR_MAX_VEC_COMPONENTS];
      struct pipe_stream_output so_output = local_so_info->output[i];
      uint32_t so_key = (uint32_t) so_output.register_index << 2 | so_output.start_component;
      struct hash_entry *he = _mesa_hash_table_search(ctx->so_outputs, &so_key);
      assert(he);
      SpvId so_output_var_id = (SpvId)(intptr_t)he->data;

      SpvId type = get_output_type(ctx, so_output.register_index, so_output.num_components);
      SpvId output = ctx->outputs[so_output.register_index];
      SpvId output_type = ctx->so_output_types[so_output.register_index];
      const struct glsl_type *out_type = ctx->so_output_gl_types[so_output.register_index];

      if (!loaded_outputs[so_output.register_index])
         loaded_outputs[so_output.register_index] = spirv_builder_emit_load(&ctx->builder, output_type, output);
      SpvId src = loaded_outputs[so_output.register_index];

      SpvId result;

      for (unsigned c = 0; c < so_output.num_components; c++) {
         components[c] = so_output.start_component + c;
         /* this is the second half of a 2 * vec4 array */
         if (ctx->stage == MESA_SHADER_VERTEX && so_output.register_index == VARYING_SLOT_CLIP_DIST1)
            components[c] += 4;
      }
```
Taking a short break here, a lot has changed. There's now code for getting both the original type of the output as well as the `xfb` output type, and special handling has been added for `gl_ClipDistance`, which is potentially a `vec8` that's represented in memory as two `vec4` values spanning two separate varying slots.

This takes care of loading the value for the variable corresponding to the `xfb` output as well as building an array of the components that are going to be emitted in the `xfb` output.

Now let's get to the ugly stuff:
```c
      /* if we're emitting a scalar or the type we're emitting matches the output's original type and we're
       * emitting the same number of components, then we can skip any sort of conversion here
       */
      if (glsl_type_is_scalar(out_type) || (type == output_type && glsl_get_length(out_type) == so_output.num_components))
         result = src;
      else {
         if (ctx->stage == MESA_SHADER_VERTEX && so_output.register_index == VARYING_SLOT_POS) {
            /* gl_Position was modified by nir_lower_clip_halfz, so we need to reverse that for streamout here:
             * 
             * opengl gl_Position.z = (vulkan gl_Position.z * 2.0) - vulkan gl_Position.w
             *
             * to do this, we extract the z and w components, perform the multiply and subtract ops, then reinsert
             */
            uint32_t z_component[] = {2};
            uint32_t w_component[] = {3};
            SpvId ftype = spirv_builder_type_float(&ctx->builder, 32);
            SpvId z = spirv_builder_emit_composite_extract(&ctx->builder, ftype, src, z_component, 1);
            SpvId w = spirv_builder_emit_composite_extract(&ctx->builder, ftype, src, w_component, 1);
            SpvId new_z = emit_binop(ctx, SpvOpFMul, ftype, z, spirv_builder_const_float(&ctx->builder, 32, 2.0));
            new_z = emit_binop(ctx, SpvOpFSub, ftype, new_z, w);
            src = spirv_builder_emit_vector_insert(&ctx->builder, type, src, new_z, 2);
         }
         /* OpCompositeExtract can only extract scalars for our use here */
         if (so_output.num_components == 1) {
            result = spirv_builder_emit_composite_extract(&ctx->builder, type, src, components, so_output.num_components);
         } else if (glsl_type_is_vector(out_type)) {
            /* OpVectorShuffle can select vector members into a differently-sized vector */
            result = spirv_builder_emit_vector_shuffle(&ctx->builder, type,
                                                             src, src,
                                                             components, so_output.num_components);
            result = emit_unop(ctx, SpvOpBitcast, type, result);
         } else {
             /* for arrays, we need to manually extract each desired member
              * and re-pack them into the desired output type
              */
             for (unsigned c = 0; c < so_output.num_components; c++) {
                uint32_t member[] = { so_output.start_component + c };
                SpvId base_type = get_glsl_type(ctx, glsl_without_array(out_type));

                if (ctx->stage == MESA_SHADER_VERTEX && so_output.register_index == VARYING_SLOT_CLIP_DIST1)
                   member[0] += 4;
                components[c] = spirv_builder_emit_composite_extract(&ctx->builder, base_type, src, member, 1);
             }
             result = spirv_builder_emit_composite_construct(&ctx->builder, type, components, so_output.num_components);
         }
      }

      spirv_builder_emit_store(&ctx->builder, so_output_var_id, result);
   }
}
```
There's five blocks here:
* Handling for cases of either a scalar value or a vec/array type containing the same total number of components that are being output to `xfb`
* Handling for `gl_Position`, which, as was previously discussed, has already been converted to Vulkan coordinates, and so now the value of this variable needs to be un-converted so the expected value can be read back
* Handling for the case of extracting a single component from a vector/array, which can be done using a single [OpCompositeExtract](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#OpCompositeExtract)
* Handling for extracting a sequence of components from a vector, which is the OpVectorShuffle from the original implementation
* Handling for extracting a sequence of components from an array, which requires manually extracting each desired array element and then re-assembling the resulting component array into the output

# And now...
Well, the tests all pass, so maybe this will be the end of it.

Just kidding.
