---
published: false
---
## Everyone Knows...

That the one true benchmark for graphics is `glxgears`. It's been the standard for 20+ years, and it's going to remain the standard for a long time to come.

## Gears Through Time

Zink has gone through a couple phases of `glxgears` performance.

Everyone remembers weird, doping `glxgears` that was getting illegal amounts of frames due to its misrendering:

[![glxgears.png](https://gitlab.freedesktop.org/mesa/mesa/uploads/f32a4cae60c4dabb90400842f0940460/Screenshot_20200625_033438.png)](https://gitlab.freedesktop.org/mesa/mesa/uploads/f32a4cae60c4dabb90400842f0940460/Screenshot_20200625_033438.png)

We salute you, old friend. 

Now, however, some number of you have become aware of the new threat posed by [heavy gears](https://gitlab.freedesktop.org/mesa/mesa/-/issues/5249) in the Mesa 21.3 release. Whereas `glxgears` is usually a lightweight, performant benchmarking tool, `heavy gears` is the opposite, chugging away at up to 20% of a single CPU core with none of the accompanying performance.

[![sadgears.png]({{site.url}}/assets/sadgears.png)]({{site.url}}/assets/sadgears.png)

Terrifying.

## What Creates Such A Monster?
The answer won't surprise you: `GL_QUADS`.

Indeed, because zink is a driver relying on the Vulkan API, only the primitive types supported by Vulkan can be directly drawn. This means any app using `GL_QUADS` is going to have a very bad time.

`glxgears` is exactly one of these apps, and (now that there's a ticket open) I was forced to take action.

## Transquadmation
