---
published: false
---
## A Quick Optimization

As I mentioned last week, I'm turning a small part of my attention now to doing some performance improvements. One of the low-hanging fruits here is adding buffer ranges; in short, this means that for buffer resources (i.e., not images), the driver tracks the ranges in memory of the buffer that have data written, which allows avoiding gpu stalls when trying to read or write from a range in the buffer that's known to not have anything written.

## util_range
The `util_range` API in gallium is extraordinarily simple. Here's the key parts:

```c
struct util_range {
   unsigned start; /* inclusive */
   unsigned end; /* exclusive */

   /* for the range to be consistent with multiple contexts: */
   simple_mtx_t write_mutex;
};

/* This is like a union of two sets. */
static inline void
util_range_add(struct pipe_resource *resource, struct util_range *range,
               unsigned start, unsigned end)
{
   if (start < range->start || end > range->end) {
      if (resource->flags & PIPE_RESOURCE_FLAG_SINGLE_THREAD_USE) {
         range->start = MIN2(start, range->start);
         range->end = MAX2(end, range->end);
      } else {
         simple_mtx_lock(&range->write_mutex);
         range->start = MIN2(start, range->start);
         range->end = MAX2(end, range->end);
         simple_mtx_unlock(&range->write_mutex);
      }
   }
}

static inline boolean
util_ranges_intersect(const struct util_range *range,
                      unsigned start, unsigned end)
{
   return MAX2(start, range->start) < MIN2(end, range->end);
}
```
When the driver writes to the buffer, `util_range_add()` is called with the range being modified. When the user then tries to map the buffer using a specified range, `util_ranges_intersect()` is called with the range being mapped to determine if a stall is needed or if it can be skipped.

## Where do these ranges get added for the buffer?
Lots of places. Here's a quick list:
* `struct pipe_context::blit`
* `struct pipe_context::clear_texture`
* `struct pipe_context::set_shader_buffers`
* `struct pipe_context::set_shader_images`
* `struct pipe_context::copy_region`
* `struct pipe_context::create_stream_output_target`
* resource unmap and flush hooks

## Some Numbers
The tests are still running to see what happens here, but I found some interesting improvements using my piglit timediff display after rebasing my branch yesterday:

![piglit-dmat.png]({{site.url}}/assets/piglit-dmat.png)

I haven't made any changes to this codepath myself (that I'm aware of), so it looks like I've pulled in some improvement that's massively cutting down the time required for the codepath that handles implicit 32bit -> 64bit conversions, and timediff picked it up.