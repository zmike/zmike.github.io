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
The problem with this methodology is that there's effecively two WSI systems active simultaneously: the Mesa DRI architecture, and the (eventual) Vulkan WSI infrastructure. Vulkan WSI isn't going to work at all if it isn't in charge of deciding things like window size, which means that the existing DRI architecture can't work, neither in the platform mode nor the software mode.

As we know, **there can be only one**.

Thus Adam has been toiling away behind the scenes, taking neither vacation nor lunch break for the past ten years in order to iterate on a more optimal solution.

The result?

## Copper
If you're a Mesa developer or just a metallurgist, you know why the name Copper was chosen.

The premise of Copper is that it's a DRI interface extension which can be used exclusively by zink to avoid any of the problem areas previously mentioned. The application will create a window, create a GL context for it, and (eventually) Vulkan WSI can figure things out by just having the window/surface passed through. This shifts all the "driving" WSI code out of DRI and into Vulkan WSI, which is much more natural.

In addition to Copper, zink can now be bound to a slight variation of the Gallium software loader to skip all the driver querying bits. There's no longer anything to query, as DRI doesn't have to make decisions anymore. It just calls through to zink normally, and zink can handle everything using the Vulkan API.

Simple and clean.

## Unfortunately
This all requires a ton of code. Looking at the two largest commits:
* `29 files changed, 1413 insertions(+), 540 deletions(-)`
* `23 files changed, 834 insertions(+), 206 deletions(-)`

Is a big yikes.

I can say with certainty that these improvements won't be landing before 2022, but eventually they will in one form or another, and then zink will become significantly more flexible.