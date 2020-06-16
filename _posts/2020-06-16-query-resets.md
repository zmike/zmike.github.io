---
published: false
---
## To start with

Reworking queries was a long process, though not nearly as long as getting `xfb` completely working. I'll be going over it in more general terms since the amount of code changed is substantial, and it's a combination of additions and deletions which makes for difficult reading in diff format on a blog post.

The first thing I started looking at was getting `vkCmdResetQueryPool` out of `zink_begin_query`. The obvious choice was to move it to `zink_create_query`; the pool should reset all its queries upon creation so that they can be used immediately. It's important to remember that this command can't be called from within a render pass, which means `zink_batch_no_rp` must be used here.

The only other place that reset is required is now at the time when a query ends in a pool, as each pool is responsible for only a single type of query.

# Fences
Now up to this point, queries were sort of just fired off and forgotten, which meant that there was no way to know whether they had completed. This was going to be a problem, as it's against Vulkan spec to destroy an in-progress query, which means they must be deferred. While violating some parts of the spec might be harmless, this part is not: destroying an in-progress `xfb` query actually crashes the Intel ANV driver.

To solve this, queries must be attached first to a given `struct zink_batch` so it becomes possible to know which command buffer they're running in. With this relation established, a fence finishing for a batch can iterate over the list of active queries for that batch, marking them as having completed. This enables query destruction to be deferred when necessary to avoid violating spec.

Unfortunately, `xfb` queries allocate specific resources in the underlying driver, and attempting to defer these types of queries can result in those resources being freed by other means, so this requires Zink to block on the corresponding batch's fence whenever deleting an active `xfb` query. The queries do know when they're active now, however, so at least there's no need to block unnecessarily.

# And then also
One of the downsides of the existing query implementation is that starting and stopping a query repeatedly without fetching the results resets the pool and discards any fetched results. Now that the reset occurs on pool creation, this is no longer the case, and the only time that results are discarded is when the pool overflows on query termination. This in itself is a questionable improvement over the existing behavior of `assert()`ing, but it's a step in the right direction.