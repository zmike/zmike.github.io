---
published: false
---
## We're Back

I spent a few days locked in a hellish battle against a software implementation of doubles for [ARB_gpu_shader_fp64](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_gpu_shader_fp64.txt) in the raging summer heat. This scholarly infographic roughly documents my progress:

![fp64-clown.png]({{site.url}}/assets/fp64-clown.png)

Instead of dwelling on it, I'm going to go back to something less brain-damaging, namely [ARB_timer_query](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_timer_query.txt). This extension provides functionality for performing time queries on the gpu, both for elapsed time and just getting regular time values.

In gallium, this is represented by `PIPE_CAP_QUERY_TIME_ELAPSED` and `PIPE_CAP_QUERY_TIMESTAMP` capabilities in a driver.