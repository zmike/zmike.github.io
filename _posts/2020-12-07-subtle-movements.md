---
published: true
---
## Alright, But Now I'm Really Back

Blogging is hard. Also, getting back on track during a major holiday week is hard.

But now things are settling, and it's time to get down to brass tacks.

And code. Brass code.

Maybe just code.

## Updates
But first, some updates.

Historically when I've missed my blogging window for an extended period, it's because I'm busy. This has been the case for the past week, but I don't have much to show for it in terms of zink enhancements. There's some work ongoing on various MRs, but probably this is a good time to revise the bold statement I'd previously made: there's now roughly two weeks (9 workdays) remaining in which it's feasible to land zink patches before the end of the year, and probably hitting GL 4.6 in mainline mesa is unrealistic. I'd be pleasantly surprised if we hit 4.0 given that we'd need to be landing a minimum of 1 new MR each day.

But there are some cool things on the horizon for zink nonetheless:
* work has begun on getting zink working with lavapipe for CI purposes
* I fixed an annoying spec-related issue that now gives better compatiblity with non-Intel drivers
* improved gl_spirv support
* further performance-related work

## And Now For Something Vaguely Interesting
Looking at the second item in the above list, there's a vague sense of discomfort that anyone working deeply with shader images will recognize.

Yes, I'm talking about the `COHERENT` qualifier.

For anyone interested in a deeper reading into this GLSL debacle, check out [this stackoverflow thread](https://stackoverflow.com/questions/56340333/glsl-about-coherent-qualifier).

TL;DR, `COHERENT` is supposed to ensure that buffer/image access across shader stages is synchronized, also known as *coherency* in this context. But then also due to GL spec wording it can simultaneously mean absolutely nothing, sort of like compiler warnings, so this is an area that generally one should avoid thinking about or delving into.

Naturally, zink is delving deep into this. And of course, Vulkan makes everything better, so this issue is 100% not an issue anymore, and everything is great.

Just kidding.

Vulkan has the exact same language in [the parts of the spec referencing this behavior](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#memory-model-informative-descriptions):
```
While GLSL (and legacy SPIR-V) applies the “coherent” decoration to variables (for historical reasons), this model treats each memory access instruction as having optional implicit availability/visibility operations.
```

`optional implicit`

Aren't explicit specifications like Vulkan great?

What happens here is that the spec has no requirement that either the application or driver actually enforces coherency across resources, meaning that if an application *optionally* decides not to bother, then it's up to the driver whether to *optionally* bother guaranteeing coherent access. If neither the application nor driver take any action to guarantee this behavior, the application won't work as expected.

[![coherency.png]({{site.url}}/assets/coherency.png)]({{site.url}}/assets/coherency.png)

To fix this on the application (zink) side, image writes in shaders need to specify `MakeTexelAvailable|NonPrivateTexel` operands, and the reads need `MakeTexelVisible|NonPrivateTexel`.
