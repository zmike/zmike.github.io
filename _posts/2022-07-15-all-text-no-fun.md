---
published: false
---
## Again.

You know what I'm about to talk about.

You knew it as soon as you opened up the page.

I've said I was done with it a number of times, but deep down we all knew it was a lie.

Let's talk about XFB.

## XFB and Zink: Recap
For newcomers to the blog, Zink has two methods of emitting streamout info:
* inlined emission, where the shader output variable and XFB data are written simultaneously
* explicit emission, where the shader output variable is written and then XFB data is written later with its own explicit variables

The former is obviously the better kind since it's simpler. But also it has another benefit: it doesn't need more variables. On the surface, it seems like this should just be the same as the first reason, namely that I don't need to run some 300 line giga-function to wrangle the XFB outputs after the shader has ended.

There shouldn't be any other reason. I've got the shader. I tell it to write some XFB data. Everything works as expected.

I know this.

You know this.

But then there's someone we didn't consult, isn't there.

[![vulkan.png](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fe/Vulkan_logo.svg/1024px-Vulkan_logo.svg.png)](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fe/Vulkan_logo.svg/1024px-Vulkan_logo.svg.png)

Obviously.

And when we do consult the spec, this seemingly-benign restriction is imposed:

> **VUID-StandaloneSpirv-Location-04916**
>> The Location decorations must be used on user-defined variables

So any user-defined variable must have a location. Seems fine. Until that restriction applies to the explicit XFB outputs. The ones that are never counted as inter-stage IO and are eliminated by the Vulkan driver's compiler for all purposes except XFB. And since locations are consumed, the restrictions for locations apply: `maxVertexOutputComponents`, `maxTessellationEvaluationOutputComponents`, and `maxGeometryOutputComponents` restrict the maximum number of locations that can be used.

And surely that would never be a problem.

## Obviously.
It's XFB, so obviously it's a problem. The standard number of locations that can be relied upon for user variables is 32, which means a total of 128 components for XFB. CTS specifically hits this in the Dragonball Z region of the test suite (Enhanced Layouts, for those of you who don't speak GLCTS) with shaders that increase in complexity at a geometric rate.

Let's take a look at one of the simpler ones from `KHR-Single-GL46.enhanced_layouts.xfb_struct_explicit_location`:

```glsl
#version 430 core
#extension GL_ARB_enhanced_layouts : require

layout(isolines, point_mode) in;

struct TestStruct {
   dmat3x4 a;
   double b;
   float c;
   dvec2 d;
};
struct OuterStruct {
    TestStruct inner_struct_a;
    TestStruct inner_struct_b;
};
layout (location = 0, xfb_offset = 0) flat out OuterStruct goku;

layout(std140, binding = 0) uniform Goku {
    TestStruct uni_goku;
};

void main()
{

    goku.inner_struct_a = uni_goku;
    goku.inner_struct_b = uni_goku;
}
```

This here is a tessellation evaluation shader that uses XFB on a very complex output type. Zink's ability to inline such types is limited, which means Goku is about to throw that spirit bomb and kill off the chances of a clean CTS run.

The reasoning is how the XFB data is emitted. Rather than get a simple piece of shader info that says "output this entire struct with XFB", Gallium instead provides the incredibly helpful `struct pipe_stream_output_info` data type:

```c
/**
 * A single output for vertex transform feedback.
 */
struct pipe_stream_output
{
   unsigned register_index:6;  /**< 0 to 63 (OUT index) */
   unsigned start_component:2; /** 0 to 3 */
   unsigned num_components:3;  /** 1 to 4 */
   unsigned output_buffer:3;   /**< 0 to PIPE_MAX_SO_BUFFERS */
   unsigned dst_offset:16;     /**< offset into the buffer in dwords */
   unsigned stream:2;          /**< 0 to 3 */
};

/**
 * Stream output for vertex transform feedback.
 */
struct pipe_stream_output_info
{
   unsigned num_outputs;
   /** stride for an entire vertex for each buffer in dwords */
   uint16_t stride[PIPE_MAX_SO_BUFFERS];

   /**
    * Array of stream outputs, in the order they are to be written in.
    * Selected components are tightly packed into the output buffer.
    */
   struct pipe_stream_output output[PIPE_MAX_SO_OUTPUTS];
};
```

Each output variable is deconstructed into the number of 32-bit components that it writes (`num_components`) with an offset (`start_component`) and a zero-indexed "location" (`register_index`) which increments sequentially for all the variables output by the shader.

In short, it has absolutely no relation to the actual shader variables, and matching them back up is a considerable amount of work.

But this is XFB, so it was always going to be terrible no matter what.

## Problems
Going back to the location disaster again, you might be wondering where the problem lies.

Let's break it down with some location analysis. According to GLSL rules, here's how locations are assigned for the output variables:

```glsl
struct TestStruct {
   dmat3x4 a; <--this is effectively dvec3[4]; a dvec3 consumes 2 locations, so 4 * 2 is 8, so this consumes locations [0,7]
   double b; <--location 8
   float c; <--location 9
   dvec2 d; <--location 10
};
struct OuterStruct {
    TestStruct inner_struct_a; <--locations [0,10]
    TestStruct inner_struct_b; <--locations [11,21]
};
```

In total, and assuming I did my calculations right, 22 locations are consumed by this struct.

And this is zink, so point size must always be emitted, which means 23 locations are now consumed. This leaves `32 - 23 = 9` locations remaining for XFB.

Given that XFB works at the component level with tight packing, this means a minimum of `2 * (3 * 4) + 2 + 1 + 2 * 2 = 31` components are needed for each struct, but there's two structs, which means it's 62 components. And ignoring all other rules for a moment, it's definitely true that locations are assigned in vec4 groupings, so a minimum of `ceil(62 / 4) = 16` locations are needed to do explicit emission of this type.

But only 9 remain.

Whoops.

## Solutions?
There's a lot of ways to fix this.

The "best" way to fix it would be to improve/overhaul the inlining detection to ensure that crazy types like this are always inlined.

That's really hard to do though, and the inlining code is already ridiculously complex to the point where I'd prefer not to ever touch it again.

The "easy" way to fix it would be to make the existing code work in this scenario without changing it. This is the approach I took, namely to decompose the struct before inlining so that the inline analysis has an easier time and can successfully inline all (or most) of the outputs.

It's much simpler (and more accurate) to inline a shader that uses an output interface with no blocks like:

```glsl
dmat3x4 a;
double b;
float c;
dvec2 d;
dmat3x4 a2;
double b2;
float c2;
dvec2 d2;
```

Thus the block splitting pass was created:

```c

static bool
split_blocks(nir_shader *nir)
{
   bool progress = false;
   bool changed = true;
   do {
      progress = false;
      nir_foreach_shader_out_variable(var, nir) {
         const struct glsl_type *base_type = glsl_without_array(var->type);
         nir_variable *members[32]; //can't have more than this without breaking NIR
         if (!glsl_type_is_struct(base_type))
            continue;
         if (!glsl_type_is_struct(var->type) || glsl_get_length(var->type) == 1)
            continue;
         if (glsl_count_attribute_slots(var->type, false) == 1)
            continue;
         unsigned offset = 0;
         for (unsigned i = 0; i < glsl_get_length(var->type); i++) {
            members[i] = nir_variable_clone(var, nir);
            members[i]->type = glsl_get_struct_field(var->type, i);
            members[i]->name = (void*)glsl_get_struct_elem_name(var->type, i);
            members[i]->data.location += offset;
            offset += glsl_count_attribute_slots(members[i]->type, false);
            nir_shader_add_variable(nir, members[i]);
         }
         nir_foreach_function(function, nir) {
            bool func_progress = false;
            if (!function->impl)
               continue;
            nir_builder b;
            nir_builder_init(&b, function->impl);
            nir_foreach_block(block, function->impl) {
               nir_foreach_instr_safe(instr, block) {
                  switch (instr->type) {
                  case nir_instr_type_deref: {
                  nir_deref_instr *deref = nir_instr_as_deref(instr);
                  if (!(deref->modes & nir_var_shader_out))
                     continue;
                  if (nir_deref_instr_get_variable(deref) != var)
                     continue;
                  if (deref->deref_type != nir_deref_type_struct)
                     continue;
                  nir_deref_instr *parent = nir_deref_instr_parent(deref);
                  if (parent->deref_type != nir_deref_type_var)
                     continue;
                  deref->modes = nir_var_shader_temp;
                  parent->modes = nir_var_shader_temp;
                  b.cursor = nir_before_instr(instr);
                  nir_ssa_def *dest = &nir_build_deref_var(&b, members[deref->strct.index])->dest.ssa;
                  nir_ssa_def_rewrite_uses_after(&deref->dest.ssa, dest, &deref->instr);
                  nir_instr_remove(&deref->instr);
                  func_progress = true;
                  break;
                  }
                  default: break;
                  }
               }
            }
            if (func_progress)
               nir_metadata_preserve(function->impl, nir_metadata_none);
         }
         var->data.mode = nir_var_shader_temp;
         changed = true;
         progress = true;
      }
   } while (progress);
   return changed;
}
```

This is an simple pass that does three things:
* walks over the shader outputs and, for every struct-type variable, splits out all the struct members into new variables at their designated location
* rewrites all derefs of the struct variable's members to instead access the new, non-struct variables
* deletes the original variable and any instructions referencing it

The result is that it's much rarer to need explicit XFB emission, and it's generally easier to debug XFB problems involving shaders with struct blocks since the blocks will be eliminated.

Now as long as Goku doesn't have any senzu beans remaining, I think this might actually be the last post I ever make about XFB.

Stay tuned next week for a more interesting and exciting post. Maybe.