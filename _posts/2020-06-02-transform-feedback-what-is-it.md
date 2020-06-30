---
layout: post
title: 'Transform Feedback: What Is It?'
published: true
---
## But really, what is it?

[open.gl](https://open.gl/feedback) explores the concept in more detail, but tl;dr it's how you can get that sweet vertex data back into a buffer (or buffers) after [vertex processing](https://www.khronos.org/opengl/wiki/Vertex_Processing) is completed, i.e., after your vertex, tesselation, and geometry shaders have shot their collective shots.

Further tl;dr as it pertains to Zink: [TRANSFORM FEEDBACK IS TERRIBLE, SO WHY ARE WE DOING IT?](http://jason-blog.jlekstrand.net/2018/10/transform-feedback-is-terrible-so-why.html) Compatibility.

Critical terminology for this post:
* `XFB` - shorthand for transform feedback
* `bench day` - that glorious day when you get to the gym and lay down in order to exercise muscles that serve no functional purpose other than letting you more fully utilize the shirts you wear
* `quads` - shorthand for GL_QUADS, which is a drawing mode that outputs quadrilaterals and isn't supported in OpenGL ES, Vulkan, or Zink, though we [fake it very convincingly](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2995) at the moment
* `NTV` - shorthand for nir_to_spirv, which is the part of Zink that translates nir (glsl) into spir-v (more on this in a future post)

## Now that we're all experts...

The goal has been to get this feature working in Zink so that we can slap on an **OpenGL 3.0 Support** sticker, as it's the only remaining item on the workboard. Original support was written by David Airlie ca. 2018, but it was a rough proof of concept that had bitrotted considerably over the years, as all code languishing in [old WIP branches](https://gitlab.freedesktop.org/kusma/mesa/-/commits/zink-old-gl3/) must. My mission, which was my first ever mission in Zink-land, was to rebase this work onto current HEAD and then smooth it out for more general usage. What followed was weeks of intermittent staring and boggling at various parts of the mesa codebase, diffs of old patches, OpenGL and Vulkan specs, and also getting some expert-level tutoring in a few matters from more experienced members of the mesa community.

## The process

...was not simple. Perhaps unwisely, I chose Marek Olšák's spec@ext_transform_feedback2@draw-auto test in piglit (more on piglit in a future post) as a starting point. This test is the equivalent of taking your driver into the gym for a full, thorough `bench day`, followed immediately by planting your driver in the squat rack until it hits a new 1RM (more on critical gym terminology in future posts). In short, it draws three differently-colored `quads`, pausing and resuming `xfb` along the way, then re-draws them in another spot using the `xfb` buffer and checks whether they match.

## How I started tackling the problem

First I threw all the WIP code into a blender and used the biological equivalent of machine learning to apply it to the current mesa tree so that it compiled. Then, feeling brash and overconfident after some high quality shitposting (more on shitposting in a future post), I tried running the test.

`assert()` errors filled my screen, and I blacked out. When I came to, my screen was still filled with errors.

## And then...

...I had to start figuring out what was going wrong. The most obvious wrong-thinging was in `ntv`, which was creating the output that triggered unlimited `assert()` errors in the spir-v compiler.

Because also: what exactly is a vector shuffle, aka OpVectorShuffle? The answer to this and more in a future post.
