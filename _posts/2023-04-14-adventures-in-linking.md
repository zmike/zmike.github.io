---
published: false
---
## First

As I mentioned a week or three ago, I deleted comments on the blog because (apparently) the widget was injecting ads. My b. I wish I could say the ad revenue was worth it, but it wasn't.

With that said, I'm looking at ways to bring comments back. I've seen a number of possibilities, but none have really grabbed me:
* [hyvor](https://hyvor.com/) looks nice, but the pricing/features seem entirely dynamic with no future guarantee what I'd actually be getting
* [disqus](https://disqus.com/) is what I was using, so I won't be going back to that
* [fastcomments](https://fastcomments.com/) might be a possibility?
* static comments baked into the site is an option with some fancy webhooks, but it already takes a couple minutes to do any sort of page deployment, and then there's the issue of spam

If anyone has other ideas, [post here](https://github.com/zmike/zmike.github.io/pull/1) about it.

## Main Topic
After this boring procedural opening, let's get to something exciting that nobody blogs about: shader linking.

**What is shader linking?** Shader linking is the process by which shaders are "linked" together to match up I/O blocks and optimize the runtime. There's a lot of rules for what compilers can and can't do during linking, and I'm sure that's all very interesting, and probably there's someone out there who would want to read about that, but we'll save that topic for another day. And another blog.

I want to talk about one part of linking in particular, and that's interface matching. Let's check out some Vulkan spec text:

```
15.1.3. Interface Matching
An output variable, block, or structure member in a given shader stage has an interface match with
an input variable, block, or structure member in a subsequent shader stage if they both adhere to
the following conditions:
• They have equivalent decorations, other than:
  ◦ XfbBuffer, XfbStride, Offset, and Stream
  ◦ one is not decorated with Component and the other is declared with a Component of 0
  ◦ Interpolation decorations
  ◦ RelaxedPrecision if one is an input variable and the other an output variable
• Their types match as follows:
  ◦ if the input is declared in a tessellation control or geometry shader as an OpTypeArray with
an Element Type equivalent to the OpType* declaration of the output, and neither is a structure
member; or
  ◦ if the maintenance4 feature is enabled, they are declared as OpTypeVector variables, and the
output has a Component Count value higher than that of the input but the same Component Type;
or
  ◦ if the output is declared in a mesh shader as an OpTypeArray with an Element Type equivalent
to the OpType* declaration of the input, and neither is a structure member; or
  ◦ if the input is decorated with PerVertexKHR, and is declared in a fragment shader as an
1240OpTypeArray with an Element Type equivalent to the OpType* declaration of the output, and
neither the input nor the output is a structure member; or
  ◦ if in any other case they are declared with an equivalent OpType* declaration.
• If both are structures and every member has an interface match.
```

Fascinating. Take a moment to digest.

## Anyway
Once again that's all very interesting, and probably there's someone out there who wanted to read about that, but this isn't quite today's topic either.

Today's topic is this one line a short ways below:

`Shaders can declare and write to output variables that are not declared or read by the subsequent stage.`

This allows e.g., a vertex shader to write an output variable that a fragment shader doesn't read. Nobody has ever seen a problem with this in Vulkan. The reason is *pipelines*. Yes, that concept about which Khronos has recently made [questionable statements](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today), courtesy of Nintendo, based on the new [VK_EXT_shader_object](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_shader_object.html) extension. In a pipeline, all the shaders get *linked*, which means the compiler can delete these unused variables. Or, if not delete, then it can at least use the explicit location info for variables to ensure that I/O is matched up properly.

And because of pipelines, everything works great.

But what happens if pipelines/linking go away?

## Uh-oh
Everyone saw this coming as soon as the blog loaded. With shader objects (and GPL fastlink), it now becomes possible to create unlinked shaders with mismatched outputs. The shader code is correct, the Vulkan API usage to create the shaders is correct, but is the execution still going to be correct?

[![cts.png]({{site.url}}/assets/cts.png)]({{site.url}}/assets/cts.png)

Right. CTS. So let's check...

[![clown/1.png]({{site.url}}/assets/clown/1.png)]({{site.url}}/assets/clown/1.png)

Okay, there's no public CTS available for `VK_EXT_shader_object` yet, but I'm sure it's coming soon.

[![clown/2.png]({{site.url}}/assets/clown/2.png)]({{site.url}}/assets/clown/2.png)

I have access to the private CTS repos, and I can see that there is (a lot of) CTS for this extension, which is a relief, and obviously I already knew this since lavapipe has passed everything, and I'm *sure* there must be testing for shader interface mismatches either there or in the GPL tests.

[![clown/3.png]({{site.url}}/assets/clown/3.png)]({{site.url}}/assets/clown/3.png)

Sure, maybe there's no tests for this, but it must be on the test plan since that's so comprehensive.

[![clown/4.png]({{site.url}}/assets/clown/4.png)]({{site.url}}/assets/clown/4.png)

Alright, so it's not in the test plan, but I can add it, and that's not a problem. In the meanwhile, since zink needs this functionality, I can just test it there, and I'm sure it'll work fine.

[![clown/5.png]({{site.url}}/assets/clown/5.png)]({{site.url}}/assets/clown/5.png)

It's more broken than [AMD's VK_EXT_robustness2 handling](https://github.com/GPUOpen-Drivers/AMDVLK/issues/321), but I'm sure it'll be easy to fix.

[![clown/6.png]({{site.url}}/assets/clown/6.png)]({{site.url}}/assets/clown/6.png)

It's nightmarishly difficult, and I wasted an entire day trying to fix `nir_assign_io_var_locations`, but I'm sure only lavapipe uses it.

[![clown/7.png]({{site.url}}/assets/clown/7.png)]({{site.url}}/assets/clown/7.png)

The Vulkan drivers affected by this issue:
* RADV
* IMG
* V3DV
* PANVK
* Lavapipe
* Turnip

Basically everyone except ANV. But also maybe ANV since the extension isn't implemented there. And probably all the proprietary drivers too since there's no CTS.

Great.

## What Exactly Is The Problem?
`nir_assign_io_var_locations` works like this:
* sort the input/output variables in a given shader using the `Location` decoration
* assign each variable an index
* increment the index based on the number of locations each variable consumes

This results in a well-ordered list of variables with proper indexing that should match up both on the input side and the output side.

Except no, not really.

Consider the following simple shader interface:

```glsl
vertex shader
layout(location = 0) in highp vec4 i_color;
layout(location = 0) out highp vec4 o_color;
void main()
{
   gl_Position = vec4(some value);
   o_color = i_color;
}

fragment shader
layout(location = 0) in highp vec4 i_color;
layout(location = 0) out highp vec4 o_color;
void main()
{
    o_color = i_color;
}
```

We expect that the vertex attribute color will propagate through to the fragment output color, and that's what happens.

Vertex shader outputs:
* `o_color`, driver_location=0

Fragment shader inputs:
* `i_color`, driver_location=0

Let's modify it slightly:

```glsl
vertex shader
layout(location = 0) in highp vec4 i_color;
layout(location = 0) out highp vec2 o_color;
layout(location = 2) out highp vec2 o_color2;
void main()
{
   gl_Position = vec4(some value);
   o_color = i_color;
}

fragment shader
layout(location = 0) in highp vec2 i_color;
layout(location = 2) in highp vec2 i_color2;
layout(location = 0) out highp vec4 o_color;
void main()
{
    o_color = vec4(i_color.xy, i_color2.xy);
}
```

Vertex shader outputs:
* `o_color`, driver_location=0
* `o_color2`, driver_location=1

Fragment shader inputs:
* `i_color`, driver_location=0
* `i_color2`, driver_location=1

No problems yet.

But what about this:

```glsl
vertex shader
layout(location = 0) in highp vec4 i_color;
layout(location = 0) out highp vec2 o_color;
layout(location = 1) out highp vec4 lol;
layout(location = 2) out highp vec2 o_color2;
void main()
{
   gl_Position = vec4(some value);
   o_color = i_color;
   lol = vec4(1.0);
}

fragment shader
layout(location = 0) in highp vec2 i_color;
layout(location = 2) in highp vec2 i_color2;
layout(location = 0) out highp vec4 o_color;
void main()
{
    o_color = vec4(i_color.xy, i_color2.xy);
}
```

In a linked pipeline this works just fine: `lol` is optimized out during linking since it isn't read by the fragment shader, and location indices are then assigned correctly. But in unlinked shader objects (and with non-LTO EXT_graphics_pipeline_library), there is no linking. Which means `lol` isn't optimized out. And what happens once `nir_assign_io_var_locations` is run?

Vertex shader outputs:
* `o_color`, driver_location=0
* `lol`, driver_location=1
* `o_color2`, driver_location=2

Fragment shader inputs:
* `i_color`, driver_location=0
* `i_color2`, driver_location=1

Tada, now the shaders are broken.

## Testing
Hopefully there will be some, but at present I've had to work around this issue in zink by creating multiple separate shader variants with different locations to ensure everything matches up.

## Fixing
I made an attempt at fixing this, but it was unsuccessful. I then contacted the great Mesa compiler sage, Timothy Arceri, and he provided me with a history lesson from The Before Times. Apparently this NIR pass was originally written for GLSL and lived in mesa/st. Then Vulkan drivers wanted to use it, so it was moved to common code. Since all pipelines were monolithic and could do link-time optimizations, there were no problems.

But now LTO isn't always possible, and so we are here.

[![clown/7.png]({{site.url}}/assets/clown/7.png)]({{site.url}}/assets/clown/7.png)

It seems to me that the solution is to write an entirely new pass for Vulkan drivers to use, and that's all very interesting, and probably there's someone out there who wants to read about that, but this is the end of the post.