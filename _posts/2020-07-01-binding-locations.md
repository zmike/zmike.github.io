---
published: false
---
## No time to waste

So let's get down to pixels. The UBO indexing is now fixed-ish, which means moving onto the next step: setting up bindings for the UBOs.

A binding in this context is the numeric id assigned to a UBO for the purposes of accessing it from a shader, which also corresponds to the `uniform block index`. In mesa, this is the `struct nir_variable::data.binding` member of a UBO. A `load_ubo` instruction will take this value as its first parameter, which means there's a need to ensure that everything matches up just right.

## Where to start
Where I started was checking out the existing code, which assumes that `nir_variable::data.binding` is already set up correctly, since the comment in `mesa/src/compiler/nir/nir.h` for the member implies that—

Just kidding, that only applies to Vulkan drivers. In Zink, that needs to be manually set up since, at most, the value will have been incremented by 1 in the `nir_lower_uniforms_to_ubo` pass from yesterday's post.

With this in mind, it's time to check out a block from `zink_compiler.c`:
```c
   nir_foreach_variable(var, &nir->uniforms) {
      if (var->data.mode == nir_var_mem_ubo) {
         int binding = zink_binding(nir->info.stage,
                                    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
                                    var->data.binding);
         ret->bindings[ret->num_bindings].index = var->data.binding;
         ret->bindings[ret->num_bindings].binding = binding;
         ret->bindings[ret->num_bindings].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
         ret->num_bindings++;
```
This iterates over the uniform variables, which are now all wrapped in UBOs, setting up the binding table that will later be used in a [vkCreateDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateDescriptorSetLayout.html) call, which passes the bindings along to the underlying driver.

Unfortunately, as just mentioned, this assumes that `var->data.binding` is set, which it isn't.

## Ordering
A number of things need to be kept in mind to effectively assign all the binding values:
* The UBOs in this list are ordered backwards, with the zero-id UBO at the end of the list. As such, the bindings need to be generated in reverse order as compared to the uniforms list stored onto the shader.
* The `index` member of the binding table, however, is not the same as the `binding` as this determines the index of the buffer to be used with the specified UBO; if `nir_lower_uniforms_to_ubo` was run, then `index` begins at 0, but otherwise it will begin at 1.
* The point of the `binding` value is to bind the UBO itself, not variables contained in the UBO. This means that any uniform with a nonzero `data.location` can be ignored, as this indicates that it's located at an offset from the base of the UBO and will be accessed by the second parameter of the `load_ubo` instruction, the offset.

With all this in mind, the following changes can be made:
```c
   uint32_t cur_ubo = 0;
   /* UBO buffers are zero-indexed, but buffer 0 is always the one created by nir_lower_uniforms_to_ubo,
    * which means there is no buffer 0 if there are no uniforms
    */
   int ubo_index = !nir->num_uniforms;
   /* need to set up var->data.binding for UBOs, which means we need to start at
    * the "first" UBO, which is at the end of the list
    */
   foreach_list_typed_reverse(nir_variable, var, node, &nir->uniforms) {
      if (var->data.mode == nir_var_mem_ubo) {
         /* ignore variables being accessed if they aren't the base of the UBO */
         if (var->data.location)
            continue;
         var->data.binding = cur_ubo++;

         int binding = zink_binding(nir->info.stage,
                                    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
                                    var->data.binding);
         ret->bindings[ret->num_bindings].index = ubo_index++;
         ret->bindings[ret->num_bindings].binding = binding;
         ret->bindings[ret->num_bindings].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
         ret->num_bindings++;
```

## Declaring
Now that the binding values are all taken care of, the next step is to go back to the UBO declarations in `ntv`:

```c
static void
emit_ubo(struct ntv_context *ctx, struct nir_variable *var)
{
   uint32_t size = glsl_count_attribute_slots(var->type, false);
```
This is the first line of the function, and it's the only one that's important here. Zink is going to pad out every member of a UBO to the size of a `vec4` (because `PIPE_CAP_PACKED_UNIFORMS` is not set by the driver), which is what `size` here is being assigned as—the number of `vec4`s needed to declare the passed variable.

This isn't what the driver should be doing here. As with the binding table setup above, this is declaring UBOs themselves, *not* variables inside UBOs. As such, all of these variables can be ignored, but the base variable needs to be sized for the entire UBO.

Helpfully, this type is available as `struct nir_variable::interface_type` for the overall UBO type, which results in the following small changes:
```c
static void
emit_ubo(struct ntv_context *ctx, struct nir_variable *var)
{
   /* variables accessed inside a uniform block will get merged into a big
    * memory blob and accessed by offset
    */
   if (var->data.location)
      return;

   uint32_t size = glsl_count_attribute_slots(var->interface_type, false);
```
The UBO list in `ntv` also has to be walked backwards for its declarations in order to match the part from `zink_compiler.c`, but this is the only change necessary.

## Binding accomplished
Yes, that's sufficient for setting up the variables and bindings for all the UBOs.

Next time, I'll finish this with a back-to-the-basics look at loading memory from buffers using offsets, except it's in `SPIR-V` so everything is way more complicated.