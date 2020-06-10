---
published: false
---
## There's always more tests

Helpfully, mesa has a suite of very demanding unit tests, aka [piglit](https://gitlab.freedesktop.org/mesa/piglit), which work great for finding all manner of issues with drivers. While the code from the previous posts handled a number of tests, it turned out that there were still a ton of failing tests.

## What happened?

A number of things. The biggest culprits were:
* a [nir bug](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5297/) related to gl_PointSize; Zink (aka Vulkan) has no native point-size variable, so one has to be injected in order to draw points successfully. The problem here was that the variable was being injected unconditionally, which sometimes resulted in two gl_PointSize outputs, breaking handling of outputs in `xfb`.
* another [nir bug](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5329) in which `xfb` shader variables were being optimized such that packing them into the output buffers correctly was failing.

Next up, issues mapping output values back to the correct `xfb` buffer in `SPIR-V` conversion. The problem in this case is that gallium translates `struct nir_shader::info.outputs_written` (a value comprised of bitflags corresponding to the `enum gl_varying_slot` outputs) to a 0-indexed value as `struct pipe_stream_output::register_index`, when what's actually needed is `enum gl_varying_slot`, since that's what passes through the output-emitting function in `struct nir_variable::data.location`. To fix this, some fixup is done on the local copy of `struct pipe_stream_output_info` that gets stored onto the `struct zink_shader` that represents the vertex shader being used:
```c
/* check for a genuine gl_PointSize output vs one from nir_lower_point_size_mov */
static bool
check_psiz(struct nir_shader *s)
{
   nir_foreach_variable(var, &s->outputs) {
      if (var->data.location == VARYING_SLOT_PSIZ) {
         /* genuine PSIZ outputs will have this set */
         return !!var->data.explicit_location;
      }
   }
   return false;
}

/* semi-copied from iris */
static void
update_so_info(struct pipe_stream_output_info *so_info,
               uint64_t outputs_written, bool have_psiz)
{
   uint8_t reverse_map[64] = {};
   unsigned slot = 0;
   while (outputs_written) {
      int bit = u_bit_scan64(&outputs_written);
      /* PSIZ from nir_lower_point_size_mov breaks stream output, so always skip it */
      if (bit == VARYING_SLOT_PSIZ && !have_psiz)
         continue;
      reverse_map[slot++] = bit;
   }

   for (unsigned i = 0; i < so_info->num_outputs; i++) {
      struct pipe_stream_output *output = &so_info->output[i];

      /* Map Gallium's condensed "slots" back to real VARYING_SLOT_* enums */
      output->register_index = reverse_map[output->register_index];
   }
}
```
In these excerpts from `zink_compile_nir()`, the shader's variables are scanned for a genuine `gl_PointSize` variable originating from the shader instead of `NIR`, then that knowledge can be applied to skip over the faked PSIZ output when rewriting the `register_index` values.

## But this was only the start
Indeed, considerably more work was required to handle the rest of the tests, as there were failures related to packed output buffers and type conversions. It's a lot of code to go over, and it merits its own post.