---
published: true
---
## Variable Lists
Today I'm going to briefly go over a big-ish API change that's taking place as a result of a [MR from Jason Ekstrand](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5966).

As I've talked about in the past, zink does a lot of iterating over variables in shaders. Primarily, this access is limited to the first three variable lists in `struct nir_shader`:
```c
typedef struct nir_shader {
   /** list of uniforms (nir_variable) */
   struct exec_list uniforms;

   /** list of inputs (nir_variable) */
   struct exec_list inputs;

   /** list of outputs (nir_variable) */
   struct exec_list outputs;
```
The `uniforms` list is used in UBO support and texture calls, `outputs` is used in xfb, and `inputs` and `outputs` have both been featured in all my slot remapping adventures.

## Less Lists
The question asked by this MR (and the [provoking issue](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3145)) is "why do we have so many lists of variables when we could instead have one list?"

The issue has more discussion about the specifics of why, but for the purposes of zink, the question becomes "how does this affect us?"

Generally speaking, the changes needed here are simple. Instead of `nir_foreach_variable()` macro iterations over a specific list type, there's now `nir_foreach_variable_with_modes(var, shader, modes)`, which has an additional param for modes. This param is based on `enum nir_variable_mode`, which corresponds to `nir_variable::data.mode`, and it causes the macro to perform a check on each variable it iterates over:
```c
#define nir_foreach_variable_with_modes(var, shader, modes) \
   nir_foreach_variable_in_shader(var, shader) \
      if (_nir_shader_variable_has_mode(var, modes))
```
Thus, the zink loops like `nir_foreach_variable(var, &s->uniforms)` look like `nir_foreach_variable_with_modes(var, s, nir_var_uniform | nir_var_mem_ubo | nir_var_mem_ssbo)` after the change.

## Further Improvements
At some point after this lands, it'll be useful to go through the places in zink where variable iterating occurs and try to combine iterations, as each variable list iteration now iterates over every type of variable, so ideally that should be reduced to a single loop that handles all the things that each separate type-iteration used to handle in order to keep things `lightweight`.
