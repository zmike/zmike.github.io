---
published: true
---
## Something more recent

Armed with a colossal set of patches in my `zink-wip` branch and feeling again like maybe it was time to be a team player instead of charging off down the field on my own, I decided yesterday morning to check out Erik's [MR to enable ARB_depth_clamp](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5495) that's been blocked on a failing piglit test for several weeks. The extension was working, supposedly, and all this MR does is enable it for use, so how hard could this be?

Filled with hubris after bulldozing my way through UBO and primitive restart handling over the course of a week, I thought I could do anything.

## The test
The test, [spec@arb_depth_clamp@depth-clamp-range](https://gitlab.freedesktop.org/mesa/piglit/-/blob/master/tests/general/depth-clamp-range.c), was originally written by Eric Anholt over ten years ago and includes more recent updates from Roland Scheidegger to give it better coverage. Here's the key bits:
```c
	float white[3] = {1.0, 1.0, 1.0};
	float clear[3] = {0.0, 0.0, 0.0};

	glClearDepth(0.5);
	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
	glEnable(GL_DEPTH_TEST);
	glDepthFunc(GL_LEQUAL);

	glColor3fv(white);

	/* Keep in mind that the ortho projection flips near and far's signs,
	 * so 1.0 to quad()'s z maps to glDepthRange's near, and -1.0 maps to
	 * glDepthRange's far.
	 */

	glEnable(GL_DEPTH_CLAMP);

	glDepthRange(0.75, 0.0);
	quad(70, 30, 4); /* 0.75 - not drawn. */

	pass = piglit_probe_pixel_rgb(75, 35, clear) && pass;
```
This portion of the test:
* clears the depth buffer to `0.5`
* enables depth testing with less-than-or-equal
* enables depth clamping
* sets an inverted depth range
* draws a quad with an untransformed depth value of `4`
* checks the coordinates at the point of the quad to verify that the quad was discarded during depth testing

## Problem
Looking at the test here, there's the expectation that, after viewport transform, the depth will get clamped to `0.75`, tested with `<=` against the clear value `0.5`, and then it will be discarded since `0.75 <= 0.5` is false.

Contrary to my expectations, this quad got rendered instead of discarded. But why?

## Examining
I rushed to the pipeline creation to check out the `depthClampEnable` value of [VkPipelineRasterizationStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineRasterizationStateCreateInfo.html). It was enabled.

I checked `depthTestEnable` and `depthCompareOp` from [VkPipelineDepthStencilStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineDepthStencilStateCreateInfo.html) in the same place. All fine there too.

[vkCmdSetViewport](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdSetViewport.html) seemed a possible candidate as well, but `minDepth` and `maxDepth` were set with `0.75` and `0.0` as they should be.

Maybe this was some weirdness going on with the shader handling? It seemed unlikely, but I wanted to cover all the bases. I fired up a quick printf shader with:
```glsl
gl_FragColor = vec4(gl_FragCoord.z - 0.75, 0.0, 1.0, 1.0);
```
The idea was to verify that the depth was a value that would be clamped.

What I got was a nice, purple square, which was also what I expected.

## ANV then
At some point, it's important to consider the possibility that this was a driver bug.

That point was not yet, however.

I checked out the Vulkan [CTS](https://github.com/KhronosGroup/VK-GL-CTS), running the `dEQP-VK.draw.inverted_depth_ranges.*` glob of tests, as the problem I was having was isolated to inverted depth ranges.

Confoundingly, all the tests passed.

But then I read the tests, and I noticed something: **none of the CTS tests for inverted depth ranges have depth testing enabled**.

At last, it was time to consider the possibility that this was, in fact, not a problem in Zink. After a brief query on IRC, Jason Ekstrand produced a [patch](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5792) within seconds which resolved the issue.

This is the story of how I didn't fix a bug.
