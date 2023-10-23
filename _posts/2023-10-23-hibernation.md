---
published: false
---
# Almost That Time Again

As readers are no doubt aware by now, SGC goes into hibernation beginning around November, and that time is nearly upon us once more. To cap out another glorious year of ~~shitposting~~highly technical and informative blogging, I'll be attempting to put up a newsworthy post every day.

This is **Day 1**.

# Zink: No Longer A Hacky Workaround Driver
2023 has seen great strides in the zink ecosystem:
* Some games, most notably my favorite game of all time [X-Plane](https://developer.x-plane.com/2023/02/addressing-plugin-flickering/), are now shipping zink in order to have a consistent GL experience across platforms
* Zink has reached [official GL 4.6 conformance](https://www.khronos.org/conformance/adopters/conformant-products/opengl#submission_332) on [Imagination](https://blog.imaginationtech.com/imagination-gpus-now-support-opengl-4.6) GPUs and will be shipping as their GL implementation
* Zink can now run display servers for both X and Wayland, enabling full systems to exist without a native GL implementation

And there's plenty more, of course, but throughout all this progress has been one very minor, very annoying wrinkle.

`MESA_LOADER_DRIVER_OVERRIDE=zink` has to be specified in order to use zink, even if no other GL drivers exist on the system.

# Or Does It?
Over a year ago I [attempted](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/16168) to enable automatic zink loading if a native driver could not be loaded. It was a reasonable first attempt, but it had issues with driver loading in scenarios where hardware drivers were not permitted.

Work has slowly progressed in Mesa since that time, and various small changes have gradually pushed the teetering tower that is GLX/EGL in the direction anyone and everyone wanted, full stop.

The result is that on zink-enabled systems, loader environment variables will [no longer be necessary](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25640) as of the upcoming Mesa 23.3 release. If zink is your only GL driver, you will get zink rather than an automatic fallback to swrast.

I can't imagine anyone will need it, but remember that issues can be reported [here](https://gitlab.freedesktop.org/mesa/mesa/-/issues).