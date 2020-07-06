---
published: false
---
## Extensions Extensions Extensions

Yes, somehow there are still more extensions left to handle, so the work continues. This time, I'll be talking briefly about a specific part of [EXT_multiview_draw_buffers](https://www.khronos.org/registry/OpenGL/extensions/EXT/EXT_multiview_draw_buffers.txt), namely:

```
If a fragment shader writes to "gl_FragColor", DrawBuffersIndexedEXT
specifies a set of draw buffers into which the color written to
"gl_FragColor" is written. If a fragment shader writes to
gl_FragData, DrawBuffersIndexedEXT specifies a set of draw buffers
into which each of the multiple output colors defined by these
variables are separately written. If a fragment shader writes to
neither gl_FragColor nor gl_FragData, the values of the fragment
colors following shader execution are undefined, and may differ
for each fragment color.
```
The gist of this is:
* if a shader writes to `gl_FragColor`, then the value of `gl_FragColor` is output to all color attachments
* if a shader writes to `gl_FragData[n]`, then this `n` value corresponds to the indexed color attachment

## As expected, this is a problem
`SPIR-V` has no builtin for a color decoration on variables, which means that `gl_FragColor` goes through as a regular variable with no special handling. As such, there's similarly no special handling in the underlying Vulkan driver to split the output of this variable out to the various color attachments, which means that only the first color attachment will have the expected result when multiple color attachments are present.

The solution, as always, is more `NIR`.

## A quick pass
What needs to happen here is a lowering pass that transforms `gl_FragColor` into `gl_FragData[0]` and then outputs its value to newly-created `gl_FragData[1-7]` variables. At present, it looks like this:

```c
static bool
lower_fragcolor_instr(nir_intrinsic_instr *instr, nir_builder *b)
{
   nir_variable *out;
   if (instr->intrinsic != nir_intrinsic_store_deref)
      return false;

   out = nir_deref_instr_get_variable(nir_src_as_deref(instr->src[0]));
   if (out->data.location != FRAG_RESULT_COLOR || out->data.mode != nir_var_shader_out)
      return false;
   b->cursor = nir_after_instr(&instr->instr);

   nir_ssa_def *frag_color = nir_load_var(b, out);
   ralloc_free(out->name);
   out->name = ralloc_strdup(out, "gl_FragData[0]");

   /* translate gl_FragColor -> gl_FragData since this is already handled */
   out->data.location = FRAG_RESULT_DATA0;
   nir_component_mask_t writemask = nir_intrinsic_write_mask(instr);

   for (unsigned i = 1; i < 8; i++) {
      char name[16];
      snprintf(name, sizeof(name), "gl_FragData[%u]", i);
      nir_variable *out_color = nir_variable_create(b->shader, nir_var_shader_out,
                                                   glsl_vec4_type(),
                                                   name);
      out_color->data.location = FRAG_RESULT_DATA0 + i;
      out_color->data.driver_location = i;
      out_color->data.index = out->data.index;
      nir_store_var(b, out_color, frag_color, writemask);
   }
   return true;
}
```
This function is called in a nested loop that iterates over all the shader instructions, and it filters through until it gets an instruction that performs a store to `gl_FragColor`, which is denoted by the `FRAG_RESULT_COLOR` value in `nir_variable::data.location`. It loads the stored value, modifying the variable itself to `gl_FragData[0]` along the way, and then does a quick loop to create the rest of the `gl_FragData` variables, storing the loaded output.

Of particular importance here is:
```c
      out_color->data.index = out->data.index;
```
This is the handling for [gl_SecondaryFragColorEXT](https://www.khronos.org/registry/OpenGL/extensions/EXT/EXT_blend_func_extended.txt) from GLES, which needs to work as well.

There's no special handling for any of these variables, so the only other change needed is to throw an `Index` [decoration](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#_a_id_decoration_a_decoration) onto `ntv` variable handling to ensure that everything works as expected.