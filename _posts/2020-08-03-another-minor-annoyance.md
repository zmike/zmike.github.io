---
published: true
---
## Instancing

Once more going way out of order since it's fresh in my mind, today will be a brief look at `gl_InstanceID` and a problem I found there while hooking up [ARB_base_instance](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_base_instance.txt).

[gl_InstanceID](https://www.khronos.org/opengl/wiki/Vertex_Shader#Other_inputs) is an input for vertex shaders which provides the current instance being processed by the shader. It has a range of `[0, instanceCount]`, and this breaks the heck out of Vulkan.

## Vulkan Instancing
In Vulkan, this same variable instead has a range of `[firstInstance, firstInstance + instanceCount]`. This difference isn't explicitly documented anywhere, which means keen readers will need to pick up on the difference as noted between [Vulkan spec 14.7. Built-In Variables](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#interfaces-builtin-variables) and the OpenGL wiki, because it isn't even stated in the GLSL 1.40 spec (or anywhere else that I could find) what the range of this variable is.

## Fixing
Here's the generic int builtin loading function zink uses in `ntv` to handle instructions which load builtin variables:
```c
static void
emit_load_uint_input(struct ntv_context *ctx, nir_intrinsic_instr *intr, SpvId *var_id, const char *var_name, SpvBuiltIn builtin)
{
   SpvId var_type = spirv_builder_type_uint(&ctx->builder, 32);
   if (!*var_id)
      *var_id = create_builtin_var(ctx, var_type,
                                   SpvStorageClassInput,
                                   var_name,
                                   builtin);

   SpvId result = spirv_builder_emit_load(&ctx->builder, var_type, *var_id);
   assert(1 == nir_dest_num_components(intr->dest));
   store_dest(ctx, &intr->dest, result, nir_type_uint);
}
```
`var_id` here is a pointer to the `SpvId` member of `struct ntv_context` where the created variable is stored for later use, `var_name` is the name of the variable, and `builtin` is the SPIRV name of the builtin that's being loaded.

`gl_InstanceID` comes through here as `SpvBuiltInInstanceIndex`. To fix the difference in semantics here, I added a few lines:
```c
static void
emit_load_uint_input(struct ntv_context *ctx, nir_intrinsic_instr *intr, SpvId *var_id, const char *var_name, SpvBuiltIn builtin)
{
   SpvId var_type = spirv_builder_type_uint(&ctx->builder, 32);
   if (!*var_id)
      *var_id = create_builtin_var(ctx, var_type,
                                   SpvStorageClassInput,
                                   var_name,
                                   builtin);

   SpvId result = spirv_builder_emit_load(&ctx->builder, var_type, *var_id);
   assert(1 == nir_dest_num_components(intr->dest));
   if (builtin == SpvBuiltInInstanceIndex) {
      /* GL's gl_InstanceID always begins at 0, so we have to normalize with gl_BaseInstance */
      SpvId base = spirv_builder_emit_load(&ctx->builder, var_type, ctx->base_instance_var);
      result = emit_binop(ctx, SpvOpISub, var_type, result, base);
   }
   store_dest(ctx, &intr->dest, result, nir_type_uint);
}
```
Now when loading `gl_InstanceID`, `gl_BaseInstance` is also loaded and subtracted to give a zero-indexed value.
