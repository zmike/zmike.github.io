---
published: true
---
## A Slow Day

It's a very slow day, and I awoke at 4:30 with the burning need to finish tessellation shaders. Or maybe I was just burning because it's so hot out.

In either case, I had a couple remaining tests that were failing, and, upon closer inspection and a lot of squinting, I determined that the problem was the use of partial writes to patch output variables in tessellation control shaders.

With this knowledge in hand, I set about resolving the issue in the most bulldozer way possible, with no regard for how much of the code would survive once I discovered which nir pass I probably wanted to be running instead.

## Partial Writes
A partial write results from shader code that looks like this:
```glsl
vec4 somevec = vec4(1.0);
somevec.xz = vec2(0.0);
```
The second assignment is partial, meaning it only writes to some of the components. SPIR-V, however, doesn't have methods for writing to only certain components of a composite value, meaning that it requires either the full composite to be written (e.g., the first line of code above) or it requires that each component be written separately.

If you're a long-time reader of the blog, or if you're good at inferring from the tone of an article, I'm sure you know what came next.

## SPIRV
Here is a normal `ntv` function which stores a write to an output variable:
```c

static void
emit_store_deref(struct ntv_context *ctx, nir_intrinsic_instr *intr)
{
   SpvId ptr = get_src(ctx, &intr->src[0]);
   SpvId src = get_src(ctx, &intr->src[1]);

   SpvId type = get_glsl_type(ctx, nir_src_as_deref(intr->src[0])->type);
   SpvId result = emit_bitcast(ctx, type, src);
   spirv_builder_emit_store(&ctx->builder, ptr, result);
}
```

It's very simple. Simple is good. The only thing to be done is to take the source operand, cast it to the type of the output variable, then store (write) it to the output variable.

Here's where we're at now:
```c
static void
emit_store_deref(struct ntv_context *ctx, nir_intrinsic_instr *intr)
{
   SpvId ptr = get_src(ctx, &intr->src[0]);
   SpvId src = get_src(ctx, &intr->src[1]);

   const struct glsl_type *gtype = nir_src_as_deref(intr->src[0])->type;
   SpvId type = get_glsl_type(ctx, gtype);
   unsigned num_writes = util_bitcount(nir_intrinsic_write_mask(intr));
   unsigned wrmask = nir_intrinsic_write_mask(intr);
   if (num_writes != intr->num_components) {
      /* no idea what we do if this fails */
      assert(glsl_type_is_array(gtype) || glsl_type_is_vector(gtype));

      /* this is a partial write, so we have to loop and do a per-component write */
      SpvId result_type;
      SpvId member_type;
      if (glsl_type_is_vector(gtype)) {
         result_type = get_glsl_basetype(ctx, glsl_get_base_type(gtype));
         member_type = get_uvec_type(ctx, 32, 1);
      } else
         member_type = result_type = get_glsl_type(ctx, glsl_get_array_element(gtype));
      SpvId ptr_type = spirv_builder_type_pointer(&ctx->builder,
                                                  SpvStorageClassOutput,
                                                  result_type);
      for (unsigned i = 0; i < 4; i++)
         if ((wrmask >> i) & 1) {
            SpvId idx = emit_uint_const(ctx, 32, i);
            SpvId val = spirv_builder_emit_composite_extract(&ctx->builder, member_type, src, &i, 1);
            val = emit_bitcast(ctx, result_type, val);
            SpvId member = spirv_builder_emit_access_chain(&ctx->builder, ptr_type,
                                                           ptr, &idx, 1);
            spirv_builder_emit_store(&ctx->builder, member, val);
         }
      return;

   }
   SpvId result = emit_bitcast(ctx, type, src);
   spirv_builder_emit_store(&ctx->builder, ptr, result);
}
```
I've now taken this otherwise readable function and changed it to something unnecessarily complex. For cases where the component output dosen't match the written mask, this code extracts each written member of the source composite, then creates an access chain (which is basically pointer dereferencing / array/vector access in SPIRV) for the corresponding member of the output, and finally writes the bitcasted component.

## But...
But Mike, I'm sure you're asking now, shouldn't `nir_lower_io_to_scalar` handle this? Or what about `nir_lower_io` in general?

Well.

Maybe?

But there's a comment `/* TODO: add patch support */`, so it doesn't, and here we are.
