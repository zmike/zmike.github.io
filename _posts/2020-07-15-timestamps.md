---
published: true
---
## We're Back

I spent a few days locked in a hellish battle against a software implementation of doubles (64-bit floats) for [ARB_gpu_shader_fp64](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_gpu_shader_fp64.txt) in the raging summer heat. This scholarly infographic roughly documents my progress:

![fp64-clown.png]({{site.url}}/assets/fp64-clown.png)

Instead of dwelling on it, I'm going to go back to something very quick and less brain-damaging, namely [ARB_timer_query](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_timer_query.txt). This extension provides functionality for performing time queries on the gpu, both for elapsed time and just getting regular time values.

In gallium, this is represented by `PIPE_CAP_QUERY_TIME_ELAPSED` and `PIPE_CAP_QUERY_TIMESTAMP` capabilities in a driver.

## Extensions
Obviously there's Vulkan extensions to go with this. [VK_EXT_calibrated_timestamps](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_calibrated_timestamps.html) is the one that's needed, as this provides the functionality for retrieving time values directly from the gpu without going through a command buffer. There's the usual copy-and-paste dance for this during driver init, which includes:
* checking for the extension name
* checking for the feature
* enabling the feature
* enabling the extension
* checking for the needed extensions-specific features (in this case, `VK_TIME_DOMAIN_DEVICE_EXT`)

## Query Time
What followed, in this case, a lot of wrangling existing query code to remove asserts and finish some stubs where timestamp queries had been partially implemented. The most important part was this function:

```c
 static void
 timestamp_to_nanoseconds(struct zink_screen *screen, uint64_t *timestamp)
 {
    /* The number of valid bits in a timestamp value is determined by
     * the VkQueueFamilyProperties::timestampValidBits property of the queue on which the timestamp is written.
     * - 17.5. Timestamp Queries
     */
    *timestamp &= ((1ull << screen->timestampValidBits) - 1);;
    /* The number of nanoseconds it takes for a timestamp value to be incremented by 1
     * can be obtained from VkPhysicalDeviceLimits::timestampPeriod
     * - 17.5. Timestamp Queries
     */
    *timestamp *= screen->props.limits.timestampPeriod;
 }
```
All time values from the gpu are returned in "ticks", which are a unit decided by the underlying driver. Applications using Zink want nanoseconds, however, so this needs to be converted.

## Side Query
But then also there's the case where a timestamp can be retrieved directly, for example:
```c
glGetInteger64v(GL_TIMESTAMP, &time);
```
In Zink and other gallium drivers, this uses either a `struct pipe_screen` or `struct pipe_context` hook:
```c
   /**
    * Query a timestamp in nanoseconds. The returned value should match
    * PIPE_QUERY_TIMESTAMP. This function returns immediately and doesn't
    * wait for rendering to complete (which cannot be achieved with queries).
    */
   uint64_t (*get_timestamp)(struct pipe_screen *);


   /**
    * Query a timestamp in nanoseconds.  This is completely equivalent to
    * pipe_screen::get_timestamp() but takes a context handle for drivers
    * that require a context.
    */
   uint64_t (*get_timestamp)(struct pipe_context *);
```
The current implementation looks like:
```c
static uint64_t
zink_get_timestamp(struct pipe_context *pctx)
{
   struct zink_screen *screen = zink_screen(pctx->screen);
   uint64_t timestamp, deviation;
   assert(screen->have_EXT_calibrated_timestamps);
   VkCalibratedTimestampInfoEXT cti = {};
   cti.sType = VK_STRUCTURE_TYPE_CALIBRATED_TIMESTAMP_INFO_EXT;
   cti.timeDomain = VK_TIME_DOMAIN_DEVICE_EXT;
   screen->vk_GetCalibratedTimestampsEXT(screen->dev, 1, &cti, &timestamp, &deviation);
   timestamp_to_nanoseconds(screen, &timestamp);
   return timestamp;
}
```
Using the `VK_EXT_calibrated_timestamps` functionality from earlier, this is very simple and straightforward, unlike software implementations of `ARB_gpu_shader_fp64`.

It's a bit of a leap forward, but that's another [GL 3.3](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3246) feature done.