---
published: true
---
## Finally, An Introduction

Since the start of this blog, I've been going full speed ahead with minimal regard for explaining terminology or architecture. This was partly to bootstrap the blog and get some potentially interesting content out there, but I also wanted to provide some insight into how clueless I was when I started out in mesa.

If you've been confused by the previous posts, that's roughly where I was at the time when I first encountered whatever it was you that you've been reading about.

## What is Gallium?
There's a lot of [good documentation available](https://gallium.readthedocs.io/en/latest/intro.html) for it, but much of that documentation assumes that the reader already has fairly deep knowledge about graphics/rendering as well as pipeline architecture.

When I began working on mesa, I did not have that knowledge, so let's take a little time to go over some parts of the mesa tree, beginning with gallium.

Gallium is the API provided by `mesa/src/mesa/state_tracker`. `state_tracker` is a mesa dri driver implementation (like `i965` or `radeon`) which translates the `mesa/src/mesa/main` API and functionality into something a bit more flexible and easy to write drivers for. In particular, the state tracker is less immediate-mode functionality than core mesa, which enables greater optimization to be performed with e.g., batching and deduplication of repeated operations.

## What are the main components of the Gallium API?
The main [headers](https://gitlab.freedesktop.org/mesa/mesa/-/tree/master/src/gallium/include/pipe) for use with gallium drivers can be found in `mesa/src/gallium/include/pipe`. This contains:

* `struct pipe_screen` - an interface for accessing the underlying hardware/device layer, providing the `get_param()` methods for determining the capabilities (`PIPE_CAP_XYZ`) that a driver has. In Zink terms, this is the object that all Vulkan commands go through, as `struct zink_screen::dev` is the `VkDevice` used for everything.

* `struct pipe_context` - an interface created from a `struct pipe_screen` providing rendering context methods to handle managing states and surface objects as well as VBO drawing. In Zink, this is the component that ties everything together.

* `struct pipe_resource` - an object created from a `struct pipe_screen` representing some sort of buffer or texture. In Zink terms, any time an OpenGL command reads back data from a buffer or directly maps data to a texture, this is the object used.

* `struct pipe_surface` - an object created from a `struct pipe_context` representing a texture view that can be bound as a color/depth/stencil attachment to a framebuffer.

* `struct pipe_query` - an object created from a `struct pipe_context` representing a query of some sort, whether for performance or functional purposes.

* `struct pipe_fence_handle` - an object created from a `struct pipe_screen` representing a fence used for command stream synchronization.

## Other components
Aside from the main gallium API (which has tons more types than just those listed above), there's also:
* the GLSL compiler
* the NIR compiler
* the SPIR-V compiler
* the mesa utility API
* the gallium aux/utility API

These are written in a combination of C/C++/Python, and they're (mostly) all used in gallium drivers.
