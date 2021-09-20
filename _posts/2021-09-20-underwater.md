---
published: true
---
## ES: But Why
I got a request recently to fix up the [WebGL Aquarium](http://webglsamples.org/aquarium/aquarium.html) demo. I've had this bookmarked for a while since it's one of the only test cases for [GL_EXT_multisampled_render_to_texture](https://www.khronos.org/registry/OpenGL/extensions/EXT/EXT_multisampled_render_to_texture.txt) I'm aware of, at least when running Chrome in EGL mode.

Naturally, I decided to do both at once since this would be yet another extension that no native desktop driver in Mesa currently supports.

## Transient Multisampling
The idea behind this extension is that on tilers, a single-sample image can be temporarily treated as multisampled without needing the extra memory bandwidth that multisampled images require. The multisampled rendering image is transient, meaning it isn't loaded or written to, so it can be lazily allocated and then discarded.

Vulkan has similar mechanisms: `loadOp` and `storeOp` set to `VK_ATTACHMENT_LOAD_OP_DONT_CARE` with an image allocated using `ZINK_HEAP_DEVICE_LOCAL_LAZY` and `VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT`, then adding a resolve attachment for writeout. Simple enough.

Unfortunately, this doesn't quite work. 

As Danylo Piliaiev, RenderDoc master and Finder Of Bad Pixels, was quick to point out, this approach will discard any previous data the texture had, resulting in bad pixels. Lots of bad pixels, in fact.

The solution, which I hate, is that the "transient" multisampled image now gets a full-texture draw before its "transient" renderpass to initialize it, then is loaded with `VK_ATTACHMENT_LOAD_OP_LOAD`, effectively making it not transient at all.

Sorry tilers.

No frames for you.

<iframe src="https://gfycat.com/ifr/HairyScalyGecko" frameborder="0" scrolling="no" allowfullscreen="allowfullscreen" width="640" height="744">aquarium.gif</iframe>

But it does work and give solid performance (-20% or so while recording smh screen recorders git gud), so I can't be too mad.
