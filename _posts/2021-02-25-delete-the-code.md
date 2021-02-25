---
published: false
---
## A Losing Battle

For a long time, I've [tried](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5831) [very](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7134), [very](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/6272), [very](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7489), [very](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7935) hard to work around problems with NIR variables when it comes to UBOs and SSBOs.

Really, I have.

But the bottom line is that, at least for gallium-based drivers, they're unusable. They're so unreliable that it's only by sheer luck (and a considerable amount of it) that zink has worked at all until this point.

Don't believe me? Here's a list of just some of the hacks that are currently in use by zink to handle support for these descriptor types, along with the reason(s) why they're needed:

Hack | Reason It's Used | Bad Because?
-----|------|-----
iterating the list of variables backwards | this indexing vaguely matches the value used by shaders to access the descriptor|only works coincidentally for as long as nothing changes this ordering and explodes entirely with GL-SPIRV
skipping non-array variables with `data.location > 0` | these are (usually) explicit references to components of a block BO object in a shader | sometimes they're the only reference to the BO, and skipping them means the whole BO interface gets skipped
using different indexing for SSBO variables depending on whether `data.explicit_binding` is set|this (sometimes) works to fix indexing for SSBOs with random bindings and also atomic counters|the value is set randomly by other optimization passes and so it isn't actually reliable
atomic counters are identified by using `!strcmp(glsl_get_type_name(var->interface_type), "counters")`|counters get converted to SSBOs, but they require different indexing in order to be accessed correctly|c'mon.
runtime arrays (`array[]`) are randomly tacked onto SPIRV SSBO variables based on the variable type|fixes atomic counter array access and SSBO `length()` method|not actually needed most of the time

And then there's this monstrosity that's used for linking up SSBO variable indices with their instruction's access value (comments included for posterity):
```c
unsigned ssbo_idx = 0;
if (!is_ubo_array && var->data.explicit_binding &&
    (glsl_type_is_unsized_array(var->type) || glsl_get_length(var->interface_type) == 1)) {
    /* - block ssbos get their binding broken in gl_nir_lower_buffers,
     *   but also they're totally indistinguishable from lowered counter buffers which have valid bindings
     *
     * hopefully this is a counter or some other non-block variable, but if not then we're probably fucked
     */
    ssbo_idx = var->data.binding;
} else if (base >= 0)
   /* we're indexing into a ssbo array and already have the base index */
   ssbo_idx = base + i;
else {
   if (ctx->ssbo_mask & 1) {
      /* 0 index is used, iterate through the used blocks until we find the first unused one */
      for (unsigned j = 1; j < ctx->num_ssbos; j++)
         if (!(ctx->ssbo_mask & (1 << j))) {
            /* we're iterating forward through the blocks, so the first available one should be
             * what we're looking for
             */
            base = ssbo_idx = j;
            break;
         }
   } else
      /* we're iterating forward through the ssbos, so always assign 0 first */
      base = ssbo_idx = 0;
   assert(ssbo_idx < ctx->num_ssbos);
}
assert(!ctx->ssbos[ssbo_idx]);
ctx->ssbos[ssbo_idx] = var_id;
ctx->ssbo_mask |= 1 << ssbo_idx;
ctx->ssbo_vars[ssbo_idx] = var;
```
Does it work?

Amazingly, yes, it does work the majority of the time.

But is this really how we should live our lives?

## A Methodology To Live By
As the great compiler-warrior Jasonus Ekstrandimus once said, "Just Delete All The Code".

Truly this is a pivotal revelation, one that can induce many days of deep thinking, but how can it be applied to this scenario?

Today I present the latest in zink code deletion: a NIR pass that deletes all the broken variables and makes new ones.

[![bender.png]({{site.url}}/assets/bender.png)]({{site.url}}/assets/bender.png)

Let's get into it.

```c
uint32_t ssbo_used = 0;
uint32_t ubo_used = 0;
uint64_t max_ssbo_size = 0;
uint64_t max_ubo_size = 0;
bool ssbo_sizes[PIPE_MAX_SHADER_BUFFERS] = {false};

if (!shader->info.num_ssbos && !shader->info.num_ubos && !shader->num_uniforms)
   return false;
nir_function_impl *impl = nir_shader_get_entrypoint(shader);
nir_foreach_block(block, impl) {
   nir_foreach_instr(instr, block) {
      if (instr->type != nir_instr_type_intrinsic)
         continue;

      nir_intrinsic_instr *intrin = nir_instr_as_intrinsic(instr);
      switch (intrin->intrinsic) {
      case nir_intrinsic_store_ssbo:
         ssbo_used |= BITFIELD_BIT(nir_src_as_uint(intrin->src[1]));
         break;

      case nir_intrinsic_get_ssbo_size: {
         uint32_t slot = nir_src_as_uint(intrin->src[0]);
         ssbo_used |= BITFIELD_BIT(slot);
         ssbo_sizes[slot] = true;
         break;
      }
      case nir_intrinsic_ssbo_atomic_add:
      case nir_intrinsic_ssbo_atomic_imin:
      case nir_intrinsic_ssbo_atomic_umin:
      case nir_intrinsic_ssbo_atomic_imax:
      case nir_intrinsic_ssbo_atomic_umax:
      case nir_intrinsic_ssbo_atomic_and:
      case nir_intrinsic_ssbo_atomic_or:
      case nir_intrinsic_ssbo_atomic_xor:
      case nir_intrinsic_ssbo_atomic_exchange:
      case nir_intrinsic_ssbo_atomic_comp_swap:
      case nir_intrinsic_ssbo_atomic_fmin:
      case nir_intrinsic_ssbo_atomic_fmax:
      case nir_intrinsic_ssbo_atomic_fcomp_swap:
      case nir_intrinsic_load_ssbo:
         ssbo_used |= BITFIELD_BIT(nir_src_as_uint(intrin->src[0]));
         break;
      case nir_intrinsic_load_ubo:
      case nir_intrinsic_load_ubo_vec4:
         ubo_used |= BITFIELD_BIT(nir_src_as_uint(intrin->src[0]));
         break;
      default:
         break;
      }
   }
}
```
The start of the pass iterates over the instructions in the shader. All UBOs and SSBOs that are used get tagged into a bitfield of their index, and any SSBOs which have the `length()` method called are similarly tagged.
```c

nir_foreach_variable_with_modes(var, shader, nir_var_mem_ssbo | nir_var_mem_ubo) {
   const struct glsl_type *type = glsl_without_array(var->type);
   if (type_is_counter(type))
      continue;
   unsigned size = glsl_count_attribute_slots(type, false);
   if (var->data.mode == nir_var_mem_ubo)
      max_ubo_size = MAX2(max_ubo_size, size);
   else
      max_ssbo_size = MAX2(max_ssbo_size, size);
   var->data.mode = nir_var_shader_temp;
}
nir_fixup_deref_modes(shader);
NIR_PASS_V(shader, nir_remove_dead_variables, nir_var_shader_temp, NULL);
optimize_nir(shader);
```

Next, the existing SSBO and UBO variables get iterated over. A maximum size is stored for each type, and then the variable mode is set to temp so it can be deleted. These variables aren't actually used by the shader anymore, so this is definitely okay.

Boom.

```c
if (!ssbo_used && !ubo_used)
   return false;
```

Early return if it turns out that there's not actually any UBO or SSBO use in the shader, and all the variables are gone to boot.

```c
struct glsl_struct_field *fields = rzalloc_array(shader, struct glsl_struct_field, 2);
fields[0].name = ralloc_strdup(shader, "base");
fields[1].name = ralloc_strdup(shader, "unsized");
```
The new variables are all going to be the same type, one which matches what's actually used during SPIRV translation: a simple struct containing an array of uints, aka `base`. SSBO variables which need the `length()` method will get a second struct member that's a runtime array, aka `unsized`.
```c
if (ubo_used) {
   const struct glsl_type *ubo_type = glsl_array_type(glsl_uint_type(), max_ubo_size * 4, 4);
   fields[0].type = ubo_type;
   u_foreach_bit(slot, ubo_used) {
      char buf[64];
      snprintf(buf, sizeof(buf), "ubo_slot_%u", slot);
      nir_variable *var = nir_variable_create(shader, nir_var_mem_ubo, glsl_struct_type(fields, 1, "struct", false), buf);
      var->interface_type = var->type;
      var->data.driver_location = slot;
   }
}
```

If there's a valid bitmask of UBOs that are used by the shader, the index slots get iterated over, and a variable is created for that slot using the same type for each one. The size is determined by the size of the biggest UBO variable that previously existed, which ensures that there won't be any errors or weirdness with access past the boundary of the variable. All the GLSL compiliation and NIR passes to this point have already handled bounds detection, so this is also fine.

```c
if (ssbo_used) {
   const struct glsl_type *ssbo_type = glsl_array_type(glsl_uint_type(), max_ssbo_size * 4, 4);
   const struct glsl_type *unsized = glsl_array_type(glsl_uint_type(), 0, 4);
   fields[0].type = ssbo_type;
   u_foreach_bit(slot, ssbo_used) {
      char buf[64];
      snprintf(buf, sizeof(buf), "ssbo_slot_%u", slot);
      if (ssbo_sizes[slot])
         fields[1].type = unsized;
      else
         fields[1].type = NULL;
      nir_variable *var = nir_variable_create(shader, nir_var_mem_ssbo,
                                              glsl_struct_type(fields, 1 + !!ssbo_sizes[slot], "struct", false), buf);
      var->interface_type = var->type;
      var->data.driver_location = slot;
   }
}
```

SSBOs are almost the same, but as previously mentioned, they also get a bonus member if they need the `length()` method. The GLSL compiler has already pre-computed the adjustment for the value that will be returned by `length()`, so it doesn't actually matter what the size of the variable is anymore.

And that's it! The entire encyclopedia of hacks can now be removed, and I can avoid ever having to look at any of this again.