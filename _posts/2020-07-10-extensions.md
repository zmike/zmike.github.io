---
published: true
---
## Something different

Usually I cover in-depth looks at various chunks of code I've been working on, but today it's going to be a more traditional style of modern blogging: memes and complaining.

I do a lot of work using Vulkan extensions. Feels like every other day I'm throwing another one into Zink (like when I had to add [VK_EXT_calibrated_timestamps](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_calibrated_timestamps.html) today for timer queries). To that end, I spend a lot of time reading the spec for extensions, though usually I prefer the man pages since they don't throw my browser on the ground and leave it a twitchy mess like the main spec page does:
[![chrome](https://pics.me.me/my-8gb-of-ram-taskmanager-rg-chumanity-gone26-one-chrome-tab-38468921.png)](https://pics.me.me/)

The problem with extensions and the spec is that extensions each have their own section, which is the same as the man page entry, and that's it. Extensions also modify the base text of the spec, but there's no clear way to know (that I've seen, anyway) what else is modified based on the entry that the extension gets.

As an example, check out [VK_EXT_robustness2](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_robustness2.html). This blurb for the spec clearly says *This extension also adds support for "null descriptors", where VK_NULL_HANDLE can be used instead of a valid handle. Accesses to null descriptors have well-defined behavior, and don’t rely on robustness.*

Alright, but what does that mean? Sure, I can chuck a null handle into a descriptor set, but how do I know that I'm complying with the spec other than yolo running it to see what the validator says?

Near as I can tell, the answer is to actually [search](https://lmgtfy.com/?q=VK_EXT_robustness2) for the extension name, then read the [diff of the commit](https://github.com/KhronosGroup/Vulkan-Docs/commit/5789d98a3fb3e02beb2f92aab5dd4b87d648cfc2) which officially adds the extension to the spec.

Compare this to OpenGL, where I can check out something like [ARB_timer_query](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_timer_query.txt) and see exactly what parts of the existing spec the extension is modifying and how they're being changed.

Maybe I'm missing something here; I've only been working on Zink for a bit over a month, so I'm certainly not going to claim to be an expert, but this feels like a huge step backwards in documentation comprehensiveness.
