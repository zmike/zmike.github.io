---
published: true
---
## Droppin' Em

While the blog was on break, some of you might have noticed that [VK_EXT_descriptor_buffer](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_descriptor_buffer.html) was released with my greasy fingerprints on it.

You might also have noticed that Zink didn't have a day-one MR.

The thing about Schrödinger's Specification is that it only remains constant while you're directly observing it. As soon as you look away, it becomes an amorphous entity changing in unknown and confusing ways.

But this extension does provide some nice benefits and future optimization sites for Zink:
* further reduces CPU overhead by eliminating more API calls
* allows for direct control over descriptor VRAM allocation
* avoids potentially unnecessary flushing when descriptors update

## Future Improvements
The initial work I've done handles the basic case of binding and updating descriptors. There are `n` sets for all the base descriptor types, and each one gets its own descriptor buffer. Each set of buffers belongs to a batch and gets bound when the batch's cmdbuf begins recording. Simple.

There's some improvements that can still be done, however.

The first and most obvious is to add handling for bindless descriptors. I'm not sure how useful this really is since there's so few apps leveraging bindless and I haven't noticed CPU utilization being an issue in any of them. But it's something that can be done.

The second is to (massively) cut down on VRAM usage. I was conservative in my allocation for the descriptor buffers, which means they're all sized to the maximum extents from the start, and all buffers are always allocated. This means that each batch gets five descriptor buffers, each of which consumes ~20MiB of VRAM, in order to avoid any sort of flushing/stalling later that would impact performance. Instead, however, it's possible that these buffers could be allocated more dynamically, creating only the ones that will be used and then scaling up the size based on utilization. This is more of a mobile thing, and the consumption with descriptor buffers should still be less than the base mode, so it's not a high priority.

Want to test? Try `ZINK_DESCRIPTORS=db` once [the MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/20489) lands.
