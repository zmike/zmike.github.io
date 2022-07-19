---
published: true
---
## The Future Comes

Anyone else remember way back years ago when I implemented descriptor caching for zink because I couldn't even hit 60 fps in Unigine Heaven due to extreme CPU bottlenecking? Also because I got to make incredible flowcharts to post here?

Good times.

Simpler times.

But now times have changed.

## The New Perf
As recently as a year ago I blogged about new descriptor infrastructure I was working on, specifically "lazy" descriptors. This endeavor was the culmination of a month or two of wild exploration into the problem space, trying out a number of less successful options.

But it turns out the most performant option was always going to be the stupidest one: create new descriptors on every draw and jam them into the GPU.

But why is this the case?

The answer lies in an extension that has become much more widely adopted, [VK_KHR_descriptor_update_template](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_descriptor_update_template.html). This extension allows applications to allocate a template that the driver can use to jam those buffers/images into descriptor sets much more efficiently than the standard `vkUpdateDescriptorSets`. When combined with extreme bucket allocating, this methodology ends up being more performant than caching.

And sometimes by significant margins.

In Minecraft, for example, you might see a 30-50% FPS increase. In a game like Tomb Raider (2013) it'll be closer to 10-20%.

But, most likely, there won't be any scenario in which FPS goes down.

So [wave farewell](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/17636) to the old code that I'll probably delete altogether in Mesa 22.3, and embrace the new code that just works better and has been undergoing heavy testing for the past year.

Welcome to the future.
