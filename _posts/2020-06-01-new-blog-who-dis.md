---
layout: post
title: New Blog Who Dis
published: true
---
After years of using one blog, then using a number of different blogs, then not blogging at all, I'm now blogging once more. Currently, the plan is to focus on [mesa](https://www.mesa3d.org) code that I'm working on, specifically (at least for now) [zink](https://www.collabora.com/news-and-blog/blog/2018/10/31/introducing-zink-opengl-implementation-vulkan/) code.

**TL;DR:** Zink is a driver that translates OpenGL to Vulkan at runtime. It is not as fast as OpenGL, and it will never be as fast, but ideally it will be "fast enough" that it can be used for general purpose GL support, enabling driver authors for new and existing hardware to focus exclusively on supporting Vulkan rather than requiring separate drivers for both APIs.

When I began working on Zink ca. April 2020, I knew nothing about Vulkan. I'd done some bits of OpenGL code here and there (e.g., [Compiz plugin injection for Enlightenment](https://www.youtube.com/watch?v=tY6qag5KFx0&hd=1)), and I'd done some other Mesa coding (e.g., planar YUV for gallium, various IRIS-related work) but nothing on the scale of "make this moderately-functional driver pass 28,000 piglit tests".

The time is now June 2020, and much progress has been made. Stay tuned for what I hope will be daily updates of what I'm working on in Zink as well as some very in-depth explanations of real problems and solutions I encounter.
