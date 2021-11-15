---
published: false
---
## Copper: It's A Thing (Sort of)

Over the past months, I've been working with Adam "X Does What I Say" Jackson to try and improve zink's path through the arcane system of chutes and ladders that comprises Gallium's loader architecture. The recent victory in getting a Wayland system compositor running is the culmination of those efforts.

I wanted to write at least a short blog post detailing some of the Gallium changes that were necessary to make this happen, if only so I have something to refer back to when I inevevitably break things later, so let's dig in.

## Pipes: How Do They Work?
It's questionable to me whether anyone really knows how all the Gallium loader and DRI frontend interfaces work without first taking a deep read of the code and then having a nice, refreshing run around the block screaming to let all the crazy out. From what I understand of it, there's the DRI (userspace) interface, which is used by EGL/GBM/GLX/SMH to manage buffers for scanout. DRI itself is split between software and platform; each DRI interface is a composite made of all the "extensions" which provide additional functionality to enable various API-level extensions.

It's a real disaster to have to work with, and ideally the eventual removal of classic drivers will allow it to be simplified so that mere humans like me can comprehend its majesty.

Beyond all this, however, there's the notion that the DRI frontend is responsible for determining the size of the scanout buffer as well as various other attributes. The software path through this is nice and simple since there's no hardware to negotiate with, and the platform path exists.

Currently, zink runs on the platform path, which means that the DRI frontend is what "runs" zink. It chooses the framebuffer size, manages resizes, and handles multisample resolve blits as needed for every frame that gets rendered.

## Too Many ~~Cooks~~Pipes
