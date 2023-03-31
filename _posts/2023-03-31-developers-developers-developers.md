---
published: false
---
## A New Frontier

As everyone expects, Khronos has recently done a [weekly spec update for Vulkan](https://github.com/KhronosGroup/Vulkan-Docs/commit/ce847fd14cc3a81751329352ce505501c46ce35e).

What nobody expected was that this week's update would include a mammoth extension, [VK_EXT_shader_object](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_shader_object.html).

Or that it would be developed by Nintendo?

## Cool
It's a very cool extension for Zink. Effectively, it means (unoptimized) shader variants can be generated very fast. So fast that the extension should solve all the remaining issues with shader compilation and stuttering by enabling applications (zink) to create and bind shaders directly without the need for pipeline objects.

Widespread adoption in the ecosystem will take time, but [Lavapipe has day one support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/22233) as everyone expects for all the cool new extensions that I work on.

## Footnote
Since Samuel is looming over me, I must say that it is *unlikely* RADV will have support for this landed in time for 23.1, though there is an implementation in the works which passes all of CTS. A lot of refactoring is involved. Like, a **lot**.

But we're definitely, 100% committed to shipping GPL by default, or you've lost the game.
