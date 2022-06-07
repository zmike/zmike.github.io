---
published: false
---
## I Remembered

After my last blog post I was so exhausted I had to take a week off, but I'm back. In the course of my blog-free week, I remembered the secret to blogging: blog before I start real work for the day.

It seems obvious, but once that code starts flowing, the prose ain't coming.

## Updates
It's been a busy couple weeks.

As I mentioned, I've been working on getting zink running on an Adreno chromebook using turnip—the open source Qualcomm driver. When I first fired up a CTS run, I was at well over 5000 failures. Today, two weeks later, that number is closer to 150. Performance in SuperTuxKart, the only app I have that's supposed to run well, is a bit lacking, but it's unclear whether this is anything related to zink. Details to come in future posts.

Some time ago I [implemented dmabuf support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16224) for lavapipe. This is finally landing, assuming CI doesn't get lost at the pub on the way to ramming the patches into the repo. Enjoy running Vulkan-based display servers in style.

Also in lavapipe-land, 1.3 conformance submissions are pending. While 1.2 conformance [went through](https://www.khronos.org/conformance/adopters/conformant-products/vulkan#submission_684) and was [blogged about](https://airlied.blogspot.com/2022/05/lavapipe-vulkan-12-conformant.html) to great acclaim, this unfortunately can't be listed in main or a current release since the driver has 1.2 conformance but advertises 1.3 support. The goalpost is moving, but we've got our hustle shoes on.

## Real Work
On occasion I manage to get real work done. A couple weeks ago, in the course of working on turnip support, I discovered something terrible: Qualcomm has no 64bit shader support.

This is different from old-time ANV, where 64bit support isn't in the hardware but is still handled correctly in the backend and so everything works even if I clumsily fumble some `dmat3` types into the GPU.