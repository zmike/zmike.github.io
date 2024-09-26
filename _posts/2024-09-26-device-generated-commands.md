# Big.

While other development has been progressing, in the background I've been working on something *big*. Now, finally, I can talk about it.

[VK_EXT_device_generated_commands](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_device_generated_commands.html) is a new extension which, it's no exageration to say, is the biggest thing Vulkan has shipped since ray-tracing. I had the privilege of working with people across the industry while driving it, from both desktop and mobile hardware vendors, and despite it being EXT, we're going to see some truly broad adoption here.

Big shoutout to Patrick Doane, formerly of Activision-Blizzard and now (I think) at Deviation Games, for kickstarting this many years ago. Thanks for your work. I hope you're satisfied with the final product.

# What does this do?

DGC enables applications to record commands from shaders to then be executed directly. This means no more ping-ponging back and forth between CPU and GPU, which can help to eliminate performance bottlenecks. See also the [NV extension](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_NV_device_generated_commands.html) and [D3D12 ExecuteIndirect](https://microsoft.github.io/DirectX-Specs/d3d/IndirectDrawing.html) as prior art.

While this functionality is used in big games such as Starfield and Halo Infinite, those examples are ETOOBIG to really comprehend. Also the code is proprietary, so I can't share it publicly. Also I don't have the code.

Fortunately, I've hacked together a small demo program for people to look over to get a feel for the functionality.

[dgcgears](https://github.com/zmike/dgcgears) is a rough fork of vkgears from [mesa-demos](https://gitlab.freedesktop.org/mesa/demos) (thanks to zink's own godfather, Erik Faye-Lund for the original work!) which utilizes DGC to execute draws rather than record them directly.

Now here's where the crazy stuff starts.

# Changing shaders from shaders

EXT DGC adds the ability to change shaders from shaders. By creating an Indirect Execution Set, multiple sets of shaders can be bundled together and indexed into from within shaders. dgcgears uses a different vertex shader to draw each gear.

While the NV extension had this functionality, EXT takes it further, enabling it to be supported on all hardware.

# Shader Objects: fully supported

Another big feature of EXT DGC is that it is agnostic to pipelines vs shader objects vs whatever new stuff comes out in the future. If you prefer one over the other, you're free to go ahead and use that.

# VKD3D-proton: supported

I've [already written](https://github.com/HansKristian-Work/vkd3d-proton/pull/2135) the code, and it should land at some point.

# Drivers: supported

* [ANV](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31384)
* [Lavapipe](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31386)
* [NVK](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31394)
* [RADV](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31383)
* [Turnip](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31388)
* other drivers soon


# 3

Device. Generated. Commands.

Count 'em.
