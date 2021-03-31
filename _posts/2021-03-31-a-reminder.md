---
published: false
---
## Do As I Say, Not As Zink Does
Today, a brief post and lamentation.

I'm sure everyone is well aware of Vulkan semantics regarding array vs non-array image types: when using an array type, e.g., `VK_IMAGE_TYPE_2D` with `VkImageCreateInfo::arrayLayers > 1`, always use `array` members for accessing/copying/blitting, and when using a 3D type, always use `depth` members.

This means array types should use `baseArrayLayer` and `layerCount` for copying/blitting and `arrayPitch` for accessing subresource regions. Non-array types, specifically 3D types, should use `VkExtent3D::depth` and `VkSubresourceLayout::depthPitch`.

This is really important, as I've found out over the past week, given that this has not been handled as it should've in many places throughout the zink stack. Some drivers were cool and didn't make a big deal about it. Other drivers were more accurate and have just been failing all along.

## And Before I Forget
I was recently interviewed by [Boiling Steam](https://boilingsteam.com/zink-running-opengl-on-top-of-vulkan-interview-with-mike-blumenkrantz/), a small Linux gaming-oriented news site focused on creating original content and interviewing only the most important figures within the community (like me). If you've ever wanted to know more about the rich open source pedigree of Super Good Code, the interview goes deep into the back catalogue of how things got to this point.