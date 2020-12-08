---
published: false
---
## Counting
I keep saying this, but I feel like I'm progressively getting further away from the original goal of this blog, which was to talk about actual code that I'm writing and not just create great graphics memes. So today it's once again a return to the roots and the code that I had intended to talk about yesterday.

Gallium is a state tracker, and as part of this, it provides various features to make writing drivers easier. One of these features is that it rolls atomic counters into SSBOs, both in terms of the actual buffer resource and the changing of shader instructions to access atomic counters as though they're uint32_t values at an offset in a buffer. On the surface, and for most drivers, this is great: the driver just has to implement handling for SSBOs, and then they get counters as a bonus.

As always, however, for zink this is A Very Bad Thing.

## SPIRV
One of the challenges in zink is the ntv backend which translates the OpenGL shader into a Vulkan shader. Typically this means GLSL -> SPIRV, but there's also the [ARB_gl_spirv](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_gl_spirv.txt) extension which allows GL to have SPIRV shaders as well, meaning that zink has to do SPIRV -> SPIRV. GLSL has a certain way of working that NIR handles, but SPIRV is very different, and so the information that is provided by NIR for GLSL is very different than what's available for SPIRV.

In particular, SPIRV shaders have valid `binding` values for shader buffers. GLSL shaders do not. This becomes a problem when trying to determine, as zink must, exactly which descriptors are on a given resource and which descriptors need to have their bindings used vs which can be ignored. Since there's no way to differentiate a GLSL shader from a SPIRV shader, this is a challenge. It's further a challenge given that one of the NIR passes that changes shader instructions over from variable pointer derefs to explicit `block_id` and `offset` values happens to break bindings in such a way that it becomes impossible to accurately tell which SSBO variables are counters and which are actual SSBOs.

## Enter The Dark Arts
```c
if (!strcmp(glsl_get_type_name(var->interface_type), "counters"))
```

Yup. The original SSBO/counter implementation has to use `strcmp` to check the name of the variable's interface in order to accurately determine whether it's a counter-turned-SSBO.

There's also some extremely gross code in ntv for trying to match up the SSBO to its expected `block_id` based on this, but SGC is a SFW blog, so I'm going to refrain from posting it.

## Improvements
As always, there's ways to improve my code. This way came some time after I'd written SSBO support in the form of a new pipe cap, `PIPE_CAP_NIR_ATOMICS_AS_DEREF`. What this does is allow a driver to skip the Gallium code that transforms counters into SSBOs, making them very easy to detect.

With this in my pocket, I was already 5% of the way to better atomic counter handling.

The next step was to unbreak counter `location` values. The `location` is, in ntv language, the offset of a variable inside a given buffer block using a type-based unit, e.g., `location=1` would mean an offset of 4 bytes for an int type block. Here's a NIR pass I wrote to tackle the problem:

```c
static bool
fixup_counter_locations(nir_shader *shader)
{
   unsigned last_binding = 0;
   unsigned last_location = 0;
   if (!shader->info.num_abos)
      return false;
   nir_foreach_variable_with_modes(var, shader, nir_var_uniform) {
      if (!type_is_counter(var->type))
         continue;
      var->data.binding += shader->info.num_ssbos;
      if (var->data.binding != last_binding) {
         last_binding = var->data.binding;
         last_location = 0;
      }
      var->data.location = last_location++;
   }
   return true;
}

```
The premise here is that counters get merged into buffers based on their binding value, and any number of counters can exist for a given binding. Since Gallium always puts counter buffers after SSBOs, the binding used needs to be incremented by the number of real SSBOs present. With this done, all counters with matching bindings can be assumed to exist sequentially on the same buffer.

Next comes the actual SPIRV variable construction. With the knowledge that zink will be receiving some sort of NIR shader instruction like `vec1 32 ssa_0 = deref_var &some_counter`, where `some_counter` is actually a value at an offset inside a buffer, it's important to be considering how to conveniently handle the offset. I ended up with something like this:
```c
   if (type_is_counter(var->type)) {
      SpvId padding = var->data.offset ? get_sized_uint_array_type(ctx, var->data.offset / 4) : 0;
      SpvId types[2];
      if (padding)
         types[0] = padding, types[1] = array_type;
      else
         types[0] = array_type;
      struct_type = spirv_builder_type_struct(&ctx->builder, types, 1 + !!padding);
      if (padding)
         spirv_builder_emit_member_offset(&ctx->builder, struct_type, 1, var->data.offset);
   }
```
This creates a struct containing 1-2 members:
* (optional) a padding array for the variable's offset
* the actual variable type, sized as an array<uint32>

Converting the `deref_var` instruction can then be simplified into a consistent and easy to generate `OpAccessChain`:
```c
if (type_is_counter(deref->var->type)) {
   SpvId dest_type = glsl_type_is_array(deref->var->type) ?
                     get_glsl_type(ctx, deref->var->type) :
                     get_dest_uvec_type(ctx, &deref->dest);
   SpvId ptr_type = spirv_builder_type_pointer(&ctx->builder,
                                               SpvStorageClassStorageBuffer,
                                               dest_type);
   SpvId idx[] = {
      emit_uint_const(ctx, 32, !!deref->var->data.offset),
   };
   result = spirv_builder_emit_access_chain(&ctx->builder, ptr_type, result, idx, ARRAY_SIZE(idx));
}
```
After setting up the destination type for the deref, the `OpAccessChain` is generated using a single index: for cases where the variable lies at a nonzero offset it selects the second member after the padding array, otherwise it selects the first member, which is the intended counter variable.

The rest of the atomic counter conversion was just a matter of supporting the specific counter-related instructions that would otherwise have been converted to regular atomic instructions.

As a result of these changes, zink has gone from a 75% pass rate in `ARB_gl_spirv` piglit tests all the way up to around 90%.