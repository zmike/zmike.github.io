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
It's XFB, so obviously it's a problem.