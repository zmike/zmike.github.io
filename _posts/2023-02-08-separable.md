---
published: false
---
## Another Milestone

A while ago, I blogged about how zink was leveraging `VK_EXT_graphics_pipeline_library` to avoid mid-frame shader compilation, AKA hitching. This was all good and accurate in that the code existed, it worked, and when the right paths were taken, there was no mid-frame shader compiling.

The problem, of course, is that these paths were never taken. Who could have guessed: there were bugs.

These bugs are [now fixed](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21169), however, and so there should be **no more mid-frame hitching ever with zink**.

Period.

If you don't believe me, run your games with `MESA_SHADER_CACHE_DISABLE=true` and report back.

## Except
There's this little extension I hate called [ARB_separate_shader_objects](https://registry.khronos.org/OpenGL/extensions/ARB/ARB_separate_shader_objects.txt). It allows shaders to be created *separately* and then linked together at runtime in a performant manner that doesn't hitch.

If you're thinking this sounds a lot like GPL fast-linking, you're not wrong.

If, however, you've noticed the obvious flaw in this thinking, you're also not wrong.

## Separate Shaders In Vulkan
This is not a thing, but it needs to be a thing. Specifically it needs to be a thing because our favorite triangle princess, Tomb Raider (2013), uses SSO extensively, which is [why the fps is bad](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8223).

Let's think about how this would work. GPL allows for splitting up the pipeline into four sections:
* vertex/input assembly
* non-fragment shaders
* fragment shader
* multisample/blend

Zink already uses this for normal shader programs. It creates partial pipeline libraries organized like this:
* vertex/input assembly
* all shaders
* multisample/blend

The `all shaders` library is generated asynchronously during load screens, the other two are generated on-demand, and the whole thing gets fast-linked together so quickly that there is (now) zero hitching.

Logically, supporting SSO from here should just mean expanding `all shaders` to a pair of pipeline libraries that can be created asynchronously:
* vertex shader
* fragment shader

This pair of shader libraries can then be fast-linked into the `all shaders` intermediate library, and the usual codepaths can then be taken.

It should be that easy.

Right?

## Obviously Not
Because descriptors exist. And while we all hoped I'd be able to make it a year without writing Yet Another Blog Post About Vulkan Descriptor Models, we all knew it was only a matter of time before I succumbed to the inevitable.

Zink, atop sane drivers, uses six descriptor sets:
* uniforms
* UBOs
* samplers
* SSBOs
* storage images
* bindless descriptors

This is nice since it keeps the codepaths for handling each set working at a descriptor-type level, making it easy to abstract while remaining performant\*.\
\* according to me

When doing the pipeline library split above, however, this is not possible. The infinite wisdom of the GPL spec allows for independent sets between libraries, which is to say that library A can use set X, library B can use set Y, and the resulting pipeline will use sets X and Y.

But it doesn't provide for any sort of merging of sets, which means that zink's entire descriptor architecture is effectively incompatible with this use of GPL.

## And That's Fine
I wanted to write some brand new, bespoke, easily-breakable descriptor code anyway, and this gives me the perfect excuse to orphan some breathtakingly smart code in a way that will never be detected by CI.

What GPL requires is a new descriptor layout that looks more like this:
* vertex shader descriptors
* fragment shader descriptors
* null
* null
* null
* bindless descriptors

Note that leaving null sets here enables the bindless set to remain bound between SSO pipelines and regular pipelines, and no changes whatsoever need to be made to anything related to bindless. If there's one thing I don't want to touch (but will definitely fingerpaint all over within the next day or two), it's bindless descriptor handling.

The first step of this latest Vulkan Descriptor Management Blog Post is to create new shader variants and modify the descriptor info for the corresponding variables. The set index needs to be updated as above, and then the bindings also need to be updated.

Yes, normally descriptor bindings are also calculated based on the descriptor-typing, which helps keep binding values low, but for SSO they have to be adjusted so that all descriptors for a shader can exist happily in a given set regardless of how many there are. This leads to the following awfulness:

```c
void
zink_descriptor_shader_get_binding_offsets(const struct zink_shader *shader, unsigned *offsets)
{
   offsets[ZINK_DESCRIPTOR_TYPE_UBO] = 0;
   offsets[ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW] = shader->bindings[ZINK_DESCRIPTOR_TYPE_UBO][shader->num_bindings[ZINK_DESCRIPTOR_TYPE_UBO] - 1].binding + 1;
   offsets[ZINK_DESCRIPTOR_TYPE_SSBO] = offsets[ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW] + shader->bindings[ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW][shader->num_bindings[ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW] - 1].binding + 1;
   offsets[ZINK_DESCRIPTOR_TYPE_IMAGE] = offsets[ZINK_DESCRIPTOR_TYPE_SSBO] + shader->bindings[ZINK_DESCRIPTOR_TYPE_SSBO][shader->num_bindings[ZINK_DESCRIPTOR_TYPE_SSBO] - 1].binding + 1;
}
```

Given a shader, an array of per-set offsets is passed in, which then gets initialized based on the highest binding counts of previous sets to avoid collisions. These offsets are then appplied in a shader pass that looks like this:

```c
int set = nir->info.stage == MESA_SHADER_FRAGMENT;
unsigned offsets[4];
zink_descriptor_shader_get_binding_offsets(zs, offsets);
nir_foreach_variable_with_modes(var, nir, nir_var_mem_ubo | nir_var_mem_ssbo | nir_var_uniform | nir_var_image) {
   if (var->data.bindless)
      continue;
   var->data.descriptor_set = set;
   switch (var->data.mode) {
   case nir_var_mem_ubo:
         var->data.binding = !!var->data.driver_location;
         break;
   case nir_var_uniform:
      if (glsl_type_is_sampler(glsl_without_array(var->type)))
         var->data.binding += offsets[1];
      break;
   case nir_var_mem_ssbo:
      var->data.binding += offsets[2];
      break;
   case nir_var_image:
      var->data.binding += offsets[3];
      break;
   default: break;
   }
}
```

The one stupid here is that `nir_var_uniform` includes actual uniform-type variables, so those have to be ignored.

With this done, a shader's descriptors can be considered SSO-capable.

But shaders are only half the equation, which means entirely novel and fascinating code has to be written to create `VkDescriptorLayout` objects, and `VkDescriptorSet` objects, and of course there's `VkDescriptorPool`, not to mention `VkDescriptorUpdateTemplate`...

The blog post might go on infinitely if I actually did all that, so instead I've turned to our lord and savior, [VK_EXT_descriptor_buffer](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_descriptor_buffer.html). This allows me to reuse most of the existing descriptor buffer code for creating layouts, and then I can just write my descriptor data to an arbitrary bound buffer rather than create new pools/sets/templates.

## The Tricky Part
As a veteran user of descriptors and member-with-voting-rights of the Vulkan Descriptor Bloggers Consortium, nothing described above poses the slightest challenge. The hard part of SSO handling is the actual pipeline management. Because all the precompiles are done per-shader in threads, there's no reusable objects for caching with shader variants. These shaders can have exactly one variant, the default, and anything else is effectively not possible.

After a number of iterations on the concept, I settled on having a union in the `zink_gfx_program` struct which omits all of the mechanics for managing shader variants. The "separable" `zink_gfx_program` object functions like this:
* app binds SSO program
* create separable `zink_gfx_program` when possible
* trigger async creation of normal `zink_gfx_program`
* draw with separable `zink_gfx_program`
* when `zink_gfx_program` is ready, replace separable object with real object

In this way, the precompiled SSO shaders can be fast-linked like a regular pipeline to avoid hitching, and the bespoke descriptor update path will be taken using more or less the same mechanics as the normal path to guarantee matching performance. Any non-shader-variant pipeline state changes can be handled without any new code, and everything "just works".

## I Lied, This Is The Real Tricky Part
Especially clever experts have been reading up until this point with six or seven eyebrows raised with the following thought in mind.

> Doesn't Tomb Raider (2013) use tessellation shaders?

Why yes. Yes, Tomb Raider (2013) *does* use tessellation shaders, and yes, it *does* only use them for drawing hair effects.

No, GPL **cannot** do separate pipeline libraries for tessellation shaders. Or geometry shaders, for that matter.

## What Does This Mean?
What it doesn't mean is that I've fixed hitching for Tomb Raider (2013).

At present, it's not possible to fix this using Vulkan. There is simply no way to precompile separate tessellation shaders, and so I again will say that the use of SSO is [why the fps is bad](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8223).

But the above-described handling does eliminate hitching caused by simple instances of SSO, which mitigates *some* of the hitching in Tomb Raider (2013). I'm not aware of other games that use this functionality, but if there are any, hopefully they don't use tessellation and are smoothed out by [this MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21197)*.
* Note: `ZINK_DESCRIPTORS=db` is currently required to enable this functionality

We're getting there.