---
published: true
---
## Queries: What are they?

A query in mesa is where an application asks for info from the underlying driver. There's a number of APIs related to this, but right now only the `xfb` related ones matter. Specifically, `GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN` and `GL_PRIMITIVES_GENERATED`.

* `GL_PRIMITIVES_GENERATED` is a query that, when active, tracks the total number of primitives generated.
* `GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN` is a query that, when active, tracks the number of primitives written.

The difference between the two is that not all primitives generated are written, as some may be duplicates that are culled. In Vulkan, these translate to:

* `GL_PRIMITIVES_GENERATED` = `VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_PRIMITIVES_BIT`
* `GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN` = `VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT`

It's important to note that these are very different enum namespaces; one is for pipeline statistics (`VK_QUERY_TYPE_PIPELINE_STATISTICS`), and one is a regular query type. Also, `GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN` will be internally translated to `PIPE_QUERY_PRIMITIVES_EMITTED`, which has a very different name, so that's good to keep in mind as well.

## What does this look like?

Let's check out some of the important bits. In Zink, starting a query now ends up looking like this, where we use an [extension method](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBeginQueryIndexedEXT.html) for starting a query for the `xfb` query type and the regular [vkCmdBeginQuery](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBeginQuery.html) for all other queries.
```c
if (q->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT)
   zink_screen(ctx->base.screen)->vk_CmdBeginQueryIndexedEXT(batch->cmdbuf,
                                                             q->query_pool,
                                                             q->curr_query,
                                                             flags,
                                                             q->index);
else
   vkCmdBeginQuery(batch->cmdbuf, q->query_pool, q->curr_query, flags);
```
Stopping queries is now very similar:
```c
if (q->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT)
   screen->vk_CmdEndQueryIndexedEXT(batch->cmdbuf, q->query_pool, q->curr_query, q->index);
else
   vkCmdEndQuery(batch->cmdbuf, q->query_pool, q->curr_query);
```

Then finally we have the parts for fetching the returned query data:
```c
int num_results;
if (query->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT) {
   /* this query emits 2 values */
   assert(query->curr_query <= ARRAY_SIZE(results) / 2);
   num_results = query->curr_query * 2;
   VkResult status = vkGetQueryPoolResults(screen->dev, query->query_pool,
                                           0, query->curr_query,
                                           sizeof(results),
                                           results,
                                           sizeof(uint64_t),
                                           flags);
   if (status != VK_SUCCESS)
      return false;
} else {
   assert(query->curr_query <= ARRAY_SIZE(results));
   num_results = query->curr_query;
   VkResult status = vkGetQueryPoolResults(screen->dev, query->query_pool,
                                           0, query->curr_query,
                                           sizeof(results),
                                           results,
                                           sizeof(uint64_t),
                                           flags);
   if (status != VK_SUCCESS)
      return false;
}
```
In this block, there's once again a split between `VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT` results and all other query results. In this case, however, the reason is that these queries return two result values, which means the buffer is treated a bit differently.
```c
for (int i = 0; i < num_results; ++i) {
   switch (query->type) {
   case PIPE_QUERY_PRIMITIVES_GENERATED:
      result->u32 += results[i];
      break;
   case PIPE_QUERY_PRIMITIVES_EMITTED:
      /* A query pool created with this type will capture 2 integers -
       * numPrimitivesWritten and numPrimitivesNeeded -
       * for the specified vertex stream output from the last vertex processing stage.
       * - from VK_EXT_transform_feedback spec
       */
      result->u64 += results[i];
      i++;
      break;
   }
}
```
This is where the returned results get looped over. `PIPE_QUERY_PRIMITIVES_GENERATED` returns a single 32-bit uint, which is added to the result struct using the appropriate member. The `xfb` query is now `PIPE_QUERY_PRIMITIVES_EMITTED` in internal types, and it returns a sequence of two 64-bit uints: `numPrimitivesWritten` and `numPrimitivesNeeded`. Zink only needs the first value, so it gets added into the struct.

## It's that simple
Compared to the other parts of implementing `xfb`, this was very simple and straightforward. More or less just translating the GL enum values to VK and mesa, then letting it run its course.
