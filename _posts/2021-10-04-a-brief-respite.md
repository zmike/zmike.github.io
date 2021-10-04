---
published: true
---
## I'm Bad At Blogging

I'm responsible enough to admit that I'm bad at blogging.

I've said repeatedly that I'm going to blog more often, and then I go and do the complete opposite.

I don't know why I'm like this, but here we are, and now it's time for another blog post.

## What's Been Happening

In short: not a lot.

All the features I've previously blogged about have landed, and zink is once again in "release mode" until the branchpoint next week to avoid having to rush patches in at the last second. This means probably there won't be any interesting patches at all to zink until then.

We're in a good spot though, and I'm pleased with the state of the driver for this release. You probably still won't be using it to play any OpenGL games you pick up from the Winter Steam Sale, but potentially those days aren't too far off.

With that said, I do have to actually blog about something technical for once, so let's roll the dice and see what it's going to be

## ARB_bindless_texture

We did it. We got a good roll.

This is actually a cool extension for an implementation deep dive because of how (relatively) simple Vulkan makes it to handle.

First, an overview: **What is ARB_bindless_texture?**

This is an extension used by only the most elite GL apps to enable texture streaming, namely the ability to continually add more images into the rendering pipeline either for sampling or shader write operations. An image is bound to a "handle", and from there, it can be made "resident" at any time to use it in shaders. This is different from the general GL methodology where an image must be explicitly bound to a specific slot (instead each image has its own slot), and it allows for both greater flexibility and more images to be in use at any given time.

At the implementation level, this actually amounts to three distinct features:
* the ability to track and manage unique "handles" for each image that can be made resident
* the ability to access these images from shaders
* the ability to pass these images between shader stages as normal I/O

In zink, I tackled these in the order I've listed.

## Handle Management
This wouldn't have been (as) possible without one very special, very awful Vulkan extension.

You knew this was coming.

[VK_EXT_descriptor_indexing](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_descriptor_indexing.html).

That's right, it's a requirement for this, but not for the reason you might think. Zink has no need for the impossibly large descriptorsets enabled by this extension, but I did need [the other features it provides](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorBindingFlagBits.html#_description):
* `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT` - enables binding a "bindless" descriptor set once and then performing updates on it without needing to have multiple sets
* `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` - enables invalidating deleted members of an active set and leaving them as garbage values in the descriptor set as long as they won't be accessed in shaders (don't worry, this is totally safe)
* `VK_DESCRIPTOR_BINDING_UPDATE_UNUSED_WHILE_PENDING_BIT` - enables updating members of an active set that aren't currently in use by any shaders

With these, it becomes possible to implement bindless textures using the existing Gallium convention:
* create `u_idalloc` instance to track and generate integer handle IDs
* map these handle IDs to slots in a large-ish (1024) sized descriptor array
* dynamically update the slots in the set as textures are made resident/not-resident
* return handle IDs to the `u_idalloc` pool once they are destroyed and the image is no longer in use

This creates a cycle where a handle ID is allocated, an image is bound to that slot in the descriptor array, the image can be unbound, the handle ID is deleted, and then finally the ID is recycled, all while only binding and updating a single descriptor set as draws continue.

## Shader Access
Now that the images are accessible to the GPU in the bindless descriptor array, shaders will have to be updated to read them.

In NIR, bindless instructions come in two variants:
* `nir_intrinsic_bindless_image_*`
* `nir_instr_type_tex` with `nir_tex_src_texture_handle`

These have their own unique semantics that I didn't bother to look into; I only need to do completely normal array derefs, so what I actually needed was just to rewrite them back into normal-er instructions.

For the image intrinsics, that ended up being the following snippet:
```c
nir_intrinsic_instr *instr = nir_instr_as_intrinsic(in);

nir_intrinsic_op op;
#define OP_SWAP(OP) \
case nir_intrinsic_bindless_image_##OP: \
   op = nir_intrinsic_image_deref_##OP; \
   break;


/* convert bindless intrinsics to deref intrinsics */
switch (instr->intrinsic) {
OP_SWAP(atomic_add)
OP_SWAP(atomic_and)
OP_SWAP(atomic_comp_swap)
OP_SWAP(atomic_dec_wrap)
OP_SWAP(atomic_exchange)
OP_SWAP(atomic_fadd)
OP_SWAP(atomic_fmax)
OP_SWAP(atomic_fmin)
OP_SWAP(atomic_imax)
OP_SWAP(atomic_imin)
OP_SWAP(atomic_inc_wrap)
OP_SWAP(atomic_or)
OP_SWAP(atomic_umax)
OP_SWAP(atomic_umin)
OP_SWAP(atomic_xor)
OP_SWAP(format)
OP_SWAP(load)
OP_SWAP(order)
OP_SWAP(samples)
OP_SWAP(size)
OP_SWAP(store)
default:
   return false;
}

enum glsl_sampler_dim dim = nir_intrinsic_image_dim(instr);
nir_variable *var = dim == GLSL_SAMPLER_DIM_BUF ? bindless_buffer_array : bindless_image_array;
if (!var)
   var = create_bindless_image(b->shader, dim);
instr->intrinsic = op;
b->cursor = nir_before_instr(in);
nir_deref_instr *deref = nir_build_deref_var(b, var);
if (glsl_type_is_array(var->type))
   deref = nir_build_deref_array(b, deref, nir_u2uN(b, instr->src[0].ssa, 32));
nir_instr_rewrite_src_ssa(in, &instr->src[0], &deref->dest.ssa);
```

In short, swap the intrinsic back to a regular image one, then rewrite the image src as a deref of a bindless image variable (which is just `image[1024]`). In long...it's the same thing. It's actually that simple.

The tex instruction is where things get trickier.

```c
nir_variable *var = tex->sampler_dim == GLSL_SAMPLER_DIM_BUF ? bindless_buffer_array : bindless_texture_array;
if (!var)
   var = create_bindless_texture(b->shader, tex);
b->cursor = nir_before_instr(in);
nir_deref_instr *deref = nir_build_deref_var(b, var);
if (glsl_type_is_array(var->type))
   deref = nir_build_deref_array(b, deref, nir_u2uN(b, tex->src[idx].src.ssa, 32));
nir_instr_rewrite_src_ssa(in, &tex->src[idx].src, &deref->dest.ssa);
```
This part is the same as the image rewrite: just rewriting the instruction as a deref.

This part, however, is different:

```c
unsigned needed_components = glsl_get_sampler_coordinate_components(glsl_without_array(var->type));
unsigned c = nir_tex_instr_src_index(tex, nir_tex_src_coord);
unsigned coord_components = nir_src_num_components(tex->src[c].src);
if (coord_components < needed_components) {
   nir_ssa_def *def = nir_pad_vector(b, tex->src[c].src.ssa, needed_components);
   nir_instr_rewrite_src_ssa(in, &tex->src[c].src, def);
   tex->coord_components = needed_components;
}
```
The thing about bindless textures is that by the time zink sees them, they have no dimensionality. They're just textures in an array, regardless of whether they're 1D, 2D, 3D, or arrayed. This means the variables used for derefs might not have the right number of coordinate components, or the instructions using them might not have the right number. To fix this, an extra cleanup is needed here to match up the number of components with the variable being used.

With all of that in place, basic bindless operations are working.

But wait...

## Shader I/O
This was the tricky part. According to the spec, it now becomes legal to have an `image` or a `sampler` as an input or an output in a shader.

But is it really, truly necessary to pass images between the shaders?

No. No it isn't.

```c
nir_deref_instr *src_deref = nir_src_as_deref(instr->src[0]);
nir_variable *var = nir_deref_instr_get_variable(src_deref);
if (var->data.bindless)
   return false;
if (var->data.mode != nir_var_shader_in && var->data.mode != nir_var_shader_out)
   return false;
if (!glsl_type_is_image(var->type) && !glsl_type_is_sampler(var->type))
   return false;

var->type = glsl_int64_t_type();
var->data.bindless = 1;
b->cursor = nir_before_instr(in);
nir_deref_instr *deref = nir_build_deref_var(b, var);
if (instr->intrinsic == nir_intrinsic_load_deref) {
    nir_ssa_def *def = nir_load_deref(b, deref);
    nir_instr_rewrite_src_ssa(in, &instr->src[0], def);
    nir_ssa_def_rewrite_uses(&instr->dest.ssa, def);
} else {
   nir_store_deref(b, deref, instr->src[1].ssa, nir_intrinsic_write_mask(instr));
}
nir_instr_remove(in);
nir_instr_remove(&src_deref->instr);
```

Bindless shader i/o is really just passing array indices that masquerade as images. If they're rewritten back to integer types, that all goes away, and they become regular i/o that needs no additional handling.

## Just This Once
The translation to Vulkan made everything incredibly easy. I didn't need any special hacks or corner case behavior, and I didn't have to spend time reading code from other drivers to figure out what the hell I was doing wrong. Validation even works for it!

Truly miraculous.
