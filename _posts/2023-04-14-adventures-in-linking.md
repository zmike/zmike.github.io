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

But what happens if pipelines go away?

## Uh-oh
Everyone saw this coming as soon as the blog loaded. With shader objects, it now becomes possible to create unlinked shaders with mismatched outputs. The shader code is correct still, the Vulkan API usage to create the shaders is correct, but is the execution still going to be correct?

[![cts.png]({{site.url}}/assets/cts.png)]({{site.url}}/assets/cts.png)

Right. CTS. So let's check...

Okay, 