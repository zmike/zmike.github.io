---
published: false
---
## I'm Bad At Blogging

I'm responsible enough to admit that I'm bad at blogging.

I've said repeatedly that I'm going to blog more often, and then I go and do the complete opposite.

I don't know why I'm like this, but here we are, and now it's time for another blog post.

## What's Been Happening

In short: not a lot.

All the features I've previously blogged about have landed, and zink is once again in "release mode" until the branchpoint next week to avoid having to rush patches in at the last second. This means probably there won't be any interesting patches at all to zink until then.

We're in a good spot though, and I'm pleased with the state of the driver for this release. You probably still won't be using it to play any OpenGL games you pick up from the Winter Steam Sale, but potentially those days aren't too far off.

With that said, I do have to actually blog about something technical for once, so let's roll the dice and see what it's going to be

## ARB_bindless_texture

We did it. We got a good roll.

This is actually a cool extension for an implementation deep dive because of how (relatively) simple Vulkan makes it to handle.

First, an overview: **What is ARB_bindless_texture?**

This is an extension used by only the most elite GL apps to enable texture streaming, namely the ability to continually add more images into the rendering pipeline either for sampling or shader write operations. An image is bound to a "handle", and from there, it can be made "resident" at any time to use it in shaders. This is different from the general GL methodology where an image must be explicitly bound to a specific slot (instead each image has its own slot), and it allows for both greater flexibility and more images to be in use at any given time.

At the implementation level, this actually amounts to three distinct features:
* the ability to track and manage unique "handles" for each image that can be made resident
* the ability to access these images from shaders
* the ability to pass these images between shader stages as normal I/O

In zink, I tackled these in the order I've listed, and it wouldn't have been (as) possible without one very special, very awful Vulkan extension.

You knew this was coming.

[VK_EXT_descriptor_indexing](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_descriptor_indexing.html).

That's right, it's a requirement for this, but not for the reason you might think. Zink has no need for the impossibly large descriptorsets enabled by this extension, but I did need [the other features it provides](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorBindingFlagBits.html#_description):
* `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT` - enables binding a "bindless" descriptor set once and then performing updates on it without needing to have multiple sets
* `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` - enables invalidating deleted members of an active set and leaving them as garbage values in the descriptor set as long as they won't be accessed in shaders (don't worry, this is totally safe)
* `VK_DESCRIPTOR_BINDING_UPDATE_UNUSED_WHILE_PENDING_BIT` - enables updating members of an active set that aren't currently in use by any shaders

With these, it becomes possible to implement bindless textures using the existing Gallium convention:
* create `u_idalloc` instance to track and generate integer handle IDs
* map these handle IDs to slots in a large-ish (1024) sized descriptor array
* dynamically update the slots in the set as textures are made resident/not-resident
* return handle IDs to the `u_idalloc` pool once they are destroyed and the image is no longer in use

This creates a cycle where a handle ID is allocated, an image is bound to that slot in the descriptor array, the image can be unbound, the handle ID is deleted, and then finally the ID is recycled, all while only binding and updating a single descriptor set as draws continue.