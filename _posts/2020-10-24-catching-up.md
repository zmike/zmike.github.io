---
published: false
---
## Never Seen Before

A rare Saturday post because I spent so much time this week intending to blog and then somehow not getting around to it. Let's get to the status updates, and then I'm going to dive into the more interesting of the things I worked on over the past few days.

Zink has just hit another big milestone that I've just invented: as of now, my branch is **passing 97% of piglit tests** up through GL 4.6 and ES 3.2, and it's a huge improvement from earlier in the week when I was only at around 92%. That's just over 1000 failure cases remaining out of ~41,000 tests. For perspective, a table.

| |IRIS|zink-mainline|zink-wip|
|-|---|---|---|
|**Passed Tests**|43508|21225|40190|
|**Total Tests**|43785|22296|41395|
|**Pass Rate**|99.4%|95.2%|97.1%|

As always, I happen to be running on Intel hardware, so IRIS and ANV are my reference points.

It's important to note here that I'm running piglit tests, and this is very different from CTS; put another way, I may be passing over 97% of the test cases I'm running, but that doesn't mean that zink is *conformant* for any versions of GL or ES, which may not actually be possible at present (without huge amounts of awkward hacks) given the persistent issues zink has with provoking vertex handling. I expect this situation to change in the future through the addition of more Vulkan extensions, but for now I'm just accepting that there's some areas where zink is going to misrender stuff.

## What Changed?
The biggest change that boosted the `zink-wip` pass rate was my fixing 64bit vertex attributes, which in total had been accounting for ~2000 test failures.

Vertex attributes, as we all know since we're all experts in the graphics field, are the inputs for vertex shaders, and the data types for these inputs can vary just like C data types. In particular, with GL 4.1, [ARB_vertex_attrib_64bit](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_vertex_attrib_64bit.txt) became a thing, which allows 64bit values to be passed as inputs here.

Once again, this is a problem for zink.

It comes down to the difference between GL's *implicit* handling methodology and Vulkan's *explicit* handling methodology. Consider the case of a `dvec4` data type. Conceptually, this is a data type which is 4x64bit values, requiring 32bytes of storage. A `vec4` uses 16bytes of storage, and this equates to a single "slot" or "location" within the shader inputs, as everything there is vec4-aligned. This means that, by simple arithmetic, a `dvec4` requires *two* slots for its storage, one for the first two members, and another for the second two, both consuming a single 16byte slot.

When loading a `dvec4` in GL(SL), a single variable with the first location slot is used, and the driver will automatically use the second slot when loading the second half of the value.

When loading a `dvec4` in (SPIR)Vulkan, two variables with consecutive, explicit location slots must be used, and the driver will load exactly the input location specified.

This difference requires that for any `dvec3` or `dvec4` vertex input in zink, the value and also the load have to be split along the `vec4` boundary for things to work.

Gallium already performs this split on the API side, allowing zink to already be correctly setting things up in the `VkPipeline` creation, so I wrote a NIR pass to fix things on the shader side.

## Shader Rewriting
Yes, it's been at least a week since I last wrote about a NIR pass, so it's past time that I got back into that.

Going into this, the idea here is to perform the following operations within the vertex shader:
* find all `deref` operations; `deref` is used to access variables for input and output, and so it's guaranteed that any 64bit input will first have a `deref`
* create a second variable (hereafter `B`) of size `double` (for `dvec3`) or `dvec2` (for `dvec4`) to represent the second half of the value
* alter the size of the original input variable (hereafter `A`) and its `deref` instruction (hereafter `A_deref`) type to `dvec2`; this aligns the variable (and its subsequent load) to the `vec4` boundary, which enables it to be correctly read from a single location slot
* create a second `deref` instruction for `B` (hereafter `B_deref`)
* find the `load_deref` instruction for `A_deref` (hereafter `A_load`)
* alter the number of components for `A_load` to 2, matching its new `dvec2` size
* create a second `load_deref` instruction for `B_deref` which will load the remaining components (hereafter `B_load`) 
* construct a new composite `dvec3` or `dvec4` from combining `A_load` + `B_load` to match the original load of the full value (hereafter `C_load`)
* rewrite all the original uses of `A_load` to instead use `C_load`

Simple, right?

Here we go.
```c
static bool
lower_64bit_vertex_attribs_instr(nir_builder *b, nir_instr *instr, void *data)
{
   if (instr->type != nir_instr_type_deref)
      return false;
   nir_deref_instr *deref = nir_instr_as_deref(instr);
   if (deref->deref_type != nir_deref_type_var)
      return false;
   nir_variable *var = nir_deref_instr_get_variable(deref);
   if (var->data.mode != nir_var_shader_in)
      return false;
   if (!glsl_type_is_64bit(var->type) || !glsl_type_is_vector(var->type) || glsl_get_vector_elements(var->type) < 3)
      return false;
```
First, it's necessary to filter out all the instructions that aren't what should be rewritten. As above, only `dvec3` and `dvec4` types are targeted here (`dmat*` types are reduced to `dvec` types prior to this point), so anything other than a `deref` of those is ignored.
```c
   /* create second variable for the split */
   nir_variable *var2 = nir_variable_clone(var, b->shader);
   /* split new variable into second slot */
   var2->data.driver_location++;
   nir_shader_add_variable(b->shader, var2);
```
`B` is identical to `A` except in its slot location, which will always be one greater than the slot location of `A`, so `A` can be cloned here to simplify the process of creating `B`.
```c
   unsigned total_num_components = glsl_get_vector_elements(var->type);
   /* new variable is the second half of the dvec */
   var2->type = glsl_vector_type(glsl_get_base_type(var->type), glsl_get_vector_elements(var->type) - 2);
   /* clamp original variable to a dvec2 */
   deref->type = var->type = glsl_vector_type(glsl_get_base_type(var->type), 2);
```
`A` and `B` need their types modified to not cross the `vec4`/slot boundary. `A` is always a `dvec2`, which has 2 components, and `B` will always be the remaining components.
```c
   /* create deref instr for new variable */
   b->cursor = nir_after_instr(instr);
   nir_deref_instr *deref2 = nir_build_deref_var(b, var2);
```
Now `B_deref` has been added thanks to the `nir_builder` helper function which massively simplifies the process of setting up all the instruction parameters.
```c
   nir_foreach_use_safe(use_src, &deref->dest.ssa) {
```
NIR is [SSA](https://en.wikipedia.org/wiki/Static_single_assignment_form)-based, and all uses of an SSA value are tracked for the purposes of ensuring that SSA values are truly assigned only once as well as ease of rewriting them in the case where a value needs to be modified, just as this pass is doing. This use-tracking comes along with a simple API for iterating over the uses.
```c
      nir_instr *use_instr = use_src->parent_instr;
      assert(use_instr->type == nir_instr_type_intrinsic &&
             nir_instr_as_intrinsic(use_instr)->intrinsic == nir_intrinsic_load_deref);
```
The only use of `A_deref` should be `A_load`, so really iterating over the `A_deref` uses is just a quick, easy way to get from there to the `A_load` instruction.
```c
      /* this is a load instruction for the deref, and we need to split it into two instructions that we can
       * then zip back into a single ssa def */
      nir_intrinsic_instr *intr = nir_instr_as_intrinsic(use_instr);
      /* clamp the first load to 2 64bit components */
      intr->num_components = intr->dest.ssa.num_components = 2;
```
`A_load` must be clamped to a single slot location to avoid crossing the `vec4` boundary, so this is done by changing the number of components to 2, which matches the now-changed type of `A`.
```c
      b->cursor = nir_after_instr(use_instr);
      /* this is the second load instruction for the second half of the dvec3/4 components */
      nir_intrinsic_instr *intr2 = nir_intrinsic_instr_create(b->shader, nir_intrinsic_load_deref);
      intr2->src[0] = nir_src_for_ssa(&deref2->dest.ssa);
      intr2->num_components = total_num_components - 2;
      nir_ssa_dest_init(&intr2->instr, &intr2->dest, intr2->num_components, 64, NULL);
      nir_builder_instr_insert(b, &intr2->instr);
```
This is `B_load`, which loads a number of components that matches the type of `B`. It's inserted after `A_load`, though the before/after isn't important in this case. The key is just that this instruction is added before the next one.
```c
      nir_ssa_def *def[4];
      /* createa new dvec3/4 comprised of all the loaded components from both variables */
      def[0] = nir_vector_extract(b, &intr->dest.ssa, nir_imm_int(b, 0));
      def[1] = nir_vector_extract(b, &intr->dest.ssa, nir_imm_int(b, 1));
      def[2] = nir_vector_extract(b, &intr2->dest.ssa, nir_imm_int(b, 0));
      if (total_num_components == 4)
         def[3] = nir_vector_extract(b, &intr2->dest.ssa, nir_imm_int(b, 1));
      nir_ssa_def *new_vec = nir_vec(b, def, total_num_components);
```
Now that `A_load` and `B_load` both exist and are loading the corrected number of components, these components can be extracted and reassembled into a larger type for use in the shader, specifically the original `dvec3` or `dvec4` which is being used. `nir_vector_extract` performs this extraction from a given instruction by taking an index of the value to extract, and then the composite value is created by passing the extracted components to `nir_vec` as an array.
```c
      /* use the assembled dvec3/4 for all other uses of the load */
      nir_ssa_def_rewrite_uses_after(&intr->dest.ssa, nir_src_for_ssa(new_vec), new_vec->parent_instr);
```
Since this is all SSA, the NIR helpers can be used to trivially rewrite all the uses of the loaded value from the original `A_load` instruction to now use the assembled `C_load` value. It's important that only the uses after `C_load` has been created (i.e., `nir_ssa_def_rewrite_uses_after`) are those that are rewritten, however, or else the shader will also rewrite the original `A_load` value with `C_load`, breaking the shader entirely with an SSA-impossible as well as generally-impossible `C_load = vec(C_load + B_load)` assignment.
```c
   }

   return true;
}
```
Progress has occurred, so the pass returns true to reflect that.

Now those large attributes are loaded according to Vulkan spec, and everything is great because, as expected, ANV has no bugs here.