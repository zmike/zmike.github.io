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