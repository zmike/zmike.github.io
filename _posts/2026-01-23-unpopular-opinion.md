# A Big Day For Graphics

Today is a big day for graphics. We got shiny new extensions and a new RM2026 profile, huzzah.

[VK_EXT_descriptor_heap](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_descriptor_heap.html) is huge. I mean in terms of surface area, the sheer girth of the spec, and the number of years it's been under development. Seriously, check out that contributor list. Is it the longest ever? I'm not about to do comparisons, but it might be.

So this is a big deal, and everyone is out in the streets (I assume to celebrate such a monumental leap forward), and I'm not.

All hats off. Person to person, let's talk.

# Power Overwhelming

It's true that descriptor heap is incredibly powerful. It perfectly exemplifies everything that Vulkan is: low-level, verbose, flexible. vkd3d-proton will make good use of it (eventually), as this more closely relates to the DX12 mechanics it translates. Game engines will finally have something that allows them to footgun as hard as they deserve. This functionality even maps more closely to certain types of hardware, as described by a great [gfxstrand blog post](https://gfxstrand.net/faith/blog/2022/08/descriptors-are-hard/).

There is, to my knowledge, just about nothing you can't do with `VK_EXT_descriptor_heap`. It's really, really good, and I'm proud of what the Vulkan WG has accomplished here.

But I don't like it.

# What Is This Incredibly Hot Take?

It's a risky position; I don't want anyone's takeaway to be "Mike shoots down new descriptor extension as worst idea in history". We're all smart people, and we can comprehend nuance, like the difference between rb and ab in EGL patch review (protip: if anyone ever gives you an rb, they're fucking lying because nobody can fully comprehend that code).

In short, I don't expect zink to ever move to descriptor heap. If it does, it'll be years from now as a result of taking on some other even more amazing extension which depends on heaps. Why is this, I'm sure you ask. Well, there's a few reasons:

## Code Complexity
Like all things Vulkan, "getting it right" with descriptors meant creating an API so verbose that I could write novels with fewer characters than some of the struct names. Everything is brand new, with no sharing/reuse of any existing code. As anyone who has ever stepped into an unfamiliar bit of code and thought "this is garbage, I should rewrite it all" knows too well, existing code is always the worst code--but it's also the code that works and is tied into all the other existing code. Pretty soon, attempting to parachute in a new descriptor API becomes rewriting literally everything because it's all incompatible. Great for those with time and resources to spare, not so great for everyone else.

Gone are image views, which is cool and good, except that everything else in Vulkan still uses them, meaning now all image descriptors need an extra pile of code to initialize the new structs which are used only for heaps. Hope none of that was shared between rendering and descriptor use, because now there will be rendering use and descriptor use and they are completely separate. Do I hate image views? Undoubtedly, and I like this direction, but hit me up in a few more years when I can delete them everywhere.

Shader interfaces are going to be the source of most pain. Sure, it's very possible to keep existing shader infrastructure and use the mapping API with its glorious nested structs. But now you have an extra 1000 lines of mapping API structs to juggle on top. Alternatively, you can get AI to rewrite all your shaders to use the new spirv extension and have direct heap access.

## Performance
Descriptor heap maps closer to hardware, which should enable users to get more performant execution by eliminating indirection with direct heap access. This is great. Full stop.

...Unless you're like zink, where the only way to avoid shredding 47 CPUs every time you change descriptors is to use a "sliding" offset for descriptors and update it each draw (i.e., `VK_DESCRIPTOR_MAPPING_SOURCE_HEAP_WITH_PUSH_INDEX_EXT`). Then you can't use direct heap access. Which means you're still indirecting your descriptor access (which has always been the purported perf pain point of 1.0 descriptors and EXT_descriptor_buffer). You do not pass Go, you do not collect $200. All you do is write a ton of new code.

## Opinionated Development
There's a tremendous piece of exposition outlining the reasons why EXT_descriptor_heap exists in [the proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_descriptor_heap.html#_problem_statement). None of these items are incorrect. I've even contributed to this document. If I were writing an engine from scratch, I would certainly expect to use heaps for portability reasons (i.e., in theory, it should eventually be available on all hardware).

But as flexible and powerful as descriptor heap is, there are some annoying cases where it passes the buck to the user. Specifically, I'm talking about management of the sampler heap. 1.0 descriptors and descriptor buffer just handwave away the exact hardware details, but with VK_EXT_descriptor_heap, you are now the captain of your own destiny and also the manager of exactly how the hardware is allocating its samplers. So if you're on NVIDIA, where you have exactly 256 available samplers as a hardware limit, you now have to juggle that limit yourself instead of letting the driver handle it for you.

This also applies to border colors, which has its own [note in the proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_descriptor_heap.html#_why_is_there_an_explicit_custom_border_color_registration). At an objective, high-view level, it's awesome to have such fine-grained control over the hardware. Then again, it's one more thing the driver is no longer managing.

# I Don't Have A Better Solution
That's certainly the takeaway here. I'm not saying go back to 1.0 descriptors. Nobody should do that. I'm not saying stick with descriptor buffers either. Descriptor heap has been under development since before I could legally drive, and I'm certainly not smarter than everyone who worked on it.

Maybe this is the best we'll get. Maybe the future of descriptors really is micromanaging every byte of device memory and material stored within because we haven't read every blog post in existence and don't trust driver developers to make our shit run good. Maybe OpenGL, with its drivers that "just worked" under the hood (with the caveat that you, the developer, can't be an idiot), wasn't what we all wanted.

Maybe I was wrong, and we do need like five trillion more blog posts about Vulkan descriptor models. Because releasing a new descriptor extension is definitely how you get more of those blog posts.

[I'm tired, boss.](https://knowyourmeme.com/memes/im-tired-boss)