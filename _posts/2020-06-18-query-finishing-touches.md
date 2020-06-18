---
published: true
---
## Testing Testing Testing

As always, there's more tests to run. And when the [validation layers](https://github.com/KhronosGroup/Vulkan-ValidationLayers) are enabled for Vulkan (`export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` once installed), sometimes new errors pop up.

Such was the case when running tests one day when I encountered an error about `vkCmdResetQueryPool` being called inside a render pass. Indeed, there was an incorrect usage here, and it needed to be resolved. Let's take a look at why this was happening.

## Query Active-ness
Once `vkCmdBeginQuery` is called, a query is considered *active* by Vulkan, which means they can't be destroyed until they're made inactive. Queries in the active state now have association with the command buffer (batch) that they're made active in, but this also means that a query needs to be "transferred" to a new `zink_batch` any time the current one is flushed and cycled to the next batch. This happens as below:

```c
static void
flush_batch(struct zink_context *ctx)
{
   struct zink_batch *batch = zink_curr_batch(ctx);
   if (batch->rp)
      vkCmdEndRenderPass(batch->cmdbuf);

   zink_end_batch(ctx, batch);

   ctx->curr_batch++;
   if (ctx->curr_batch == ARRAY_SIZE(ctx->batches))
      ctx->curr_batch = 0;

   zink_start_batch(ctx, zink_curr_batch(ctx));
}
```
This submits the current command buffer (batch), then switches to the next one and activates it. In the process, all active queries on the current batch are stopped, then they get started once more on the new command buffer first thing so that they continue to e.g., track primitives emitted without missing any operations.

Diving deeper, `zink_end_batch()` calls through to `zink_suspend_queries()`, which calls `end_query()`, a function I've mentioned a couple times previously. After all the reworking, here's how it looks:
```c
static void
end_query(struct zink_batch *batch, struct zink_query *q)
{
   struct zink_context *ctx = batch->ctx;
   struct zink_screen *screen = zink_screen(ctx->base.screen);
   assert(q->type != PIPE_QUERY_TIMESTAMP);
   q->active = false;
   if (q->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT)
      screen->vk_CmdEndQueryIndexedEXT(batch->cmdbuf, q->query_pool, q->curr_query, q->index);
   else
      vkCmdEndQuery(batch->cmdbuf, q->query_pool, q->curr_query);
   if (++q->curr_query == q->num_queries) {
      get_query_result(&ctx->base, (struct pipe_query*)q, false, &q->big_result);
      vkCmdResetQueryPool(batch->cmdbuf, q->query_pool, 0, q->num_queries);
      q->last_checked_query = q->curr_query = 0;
   }
}
```
As seen, the query is marked inactive (as it relates to Vulkan), ended, and then the query id is incremented. If the id reaches the max number of queries in the pool, the query grabs the current results into the inline result struct discussed previously before resetting the pool.

This is where the API misuse error was coming from, as the render pass is not destroyed until the batch is reset, which occurs the next time it's made active.

A small change to the reset block at the end of `end_query()`:
```c
static void
end_query(struct zink_batch *batch, struct zink_query *q)
{
   struct zink_context *ctx = batch->ctx;
   struct zink_screen *screen = zink_screen(ctx->base.screen);
   assert(q->type != PIPE_QUERY_TIMESTAMP);
   q->active = false;
   if (q->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT)
      screen->vk_CmdEndQueryIndexedEXT(batch->cmdbuf, q->query_pool, q->curr_query, q->index);
   else
      vkCmdEndQuery(batch->cmdbuf, q->query_pool, q->curr_query);
   if (++q->curr_query == q->num_queries) {
      if (batch->rp)
         q->needs_reset = true;
      else
        reset_pool(batch, q);
   }
}
```
Now the reset can be deferred until the query is made active again, at which point it's guaranteed to not be inside a render pass.

## And that's it
The updated query code is [awaiting review](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5533), and Zink no longer `assert()`s when an application toggles its queries too many times.
