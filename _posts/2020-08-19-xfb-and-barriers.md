---
published: true
---
## Yet Another XFB Post

Sort of.

In the course of working on some code to introduce more granular resource-based pipeline barriers for synchronization, I rediscovered some parts of the xfb implementation we have that use barriers, and this got me thinking about our barrier usage in general for buffer resources.

At present, we have exactly two places in the zink codebase where we emit pipeline barriers for buffer resources:
* when binding an xfb output buffer as a target for indirect draws (i.e., drawing transform feedback)
* when binding an xfb output buffer for stream output

That's it.

So no matter how else we may use buffer resources, whether it's as a writable SSBO in a vertex shader, or a readable SSBO in a fragment shader, or an index buffer, we never use barriers.

Instead, we mostly just flush the pipeline (vkQueueSubmit) and/or stall by waiting on a fence to work around our lack of barrier usage.

## Example
An example of this in the repo now is zink's `pipe_context::set_vertex_buffers` hook:
```c
static void
zink_set_vertex_buffers(struct pipe_context *pctx,
                        unsigned start_slot,
                        unsigned num_buffers,
                        const struct pipe_vertex_buffer *buffers)
{
   struct zink_context *ctx = zink_context(pctx);

   if (buffers) {
      for (int i = 0; i < num_buffers; ++i) {
         const struct pipe_vertex_buffer *vb = buffers + i;
         struct zink_resource *res = zink_resource(vb->buffer.resource);

         ctx->gfx_pipeline_state.bindings[start_slot + i].stride = vb->stride;
         if (res && res->needs_xfb_barrier) {
            /* if we're binding a previously-used xfb buffer, we need cmd buffer synchronization to ensure
             * that we use the right buffer data
             */
            pctx->flush(pctx, NULL, 0);
            res->needs_xfb_barrier = false;
         }
      }
      ctx->gfx_pipeline_state.hash = 0;
   }

   util_set_vertex_buffers_mask(ctx->buffers, &ctx->buffers_enabled_mask,
                                buffers, start_slot, num_buffers);
}
```
Here, the `needs_xfb_barrier` flag is set any time a resource is used as a stream output buffer for a draw, and then we force a flush (which also ends the current renderpass and forces us to start a new one) before we start the next draw in order to sync.

Full disclosure: I wrote the weird parts here, but probably nobody remembers that far back anyway.

## Improvement
Now clearly it would be better for performance reasons to not be unnecessarily flushing our command buffer or forcing a new renderpass here.

Instead, a barrier can be used. **The `needs_xfb_barrier` parts of `zink_set_vertex_buffers()` can be removed**, and now sights are set on the repo-current version of a helper function for `zink_draw_vbo()` which is used to bind vertex buffers for use:
```c
static void
zink_bind_vertex_buffers(struct zink_batch *batch, struct zink_context *ctx)
{
   VkBuffer buffers[PIPE_MAX_ATTRIBS];
   VkDeviceSize buffer_offsets[PIPE_MAX_ATTRIBS];
   const struct zink_vertex_elements_state *elems = ctx->element_state;
   for (unsigned i = 0; i < elems->hw_state.num_bindings; i++) {
      struct pipe_vertex_buffer *vb = ctx->buffers + ctx->element_state->binding_map[i];
      assert(vb);
      if (vb->buffer.resource) {
         struct zink_resource *res = zink_resource(vb->buffer.resource);
         buffers[i] = res->buffer;
         buffer_offsets[i] = vb->buffer_offset;
         zink_batch_reference_resoure(batch, res);
      } else {
         buffers[i] = zink_resource(ctx->dummy_buffer)->buffer;
         buffer_offsets[i] = 0;
      }
   }

   if (elems->hw_state.num_bindings > 0)
      vkCmdBindVertexBuffers(batch->cmdbuf, 0,
                             elems->hw_state.num_bindings,
                             buffers, buffer_offsets);
}
```
In this function, the to-be-bound vertex buffers are iterated over to populate the info for `vkCmdBindVertexBuffers()`, and a reference is also added to the batch they'll be used on. But, despite being used as vertex inputs, there's no barrier here to inform the vulkan driver of zink's synchronization needs.

A revised version of this function now looks like this:
```c

static void
zink_bind_vertex_buffers(struct zink_batch *batch, struct zink_context *ctx)
{
   VkBuffer buffers[PIPE_MAX_ATTRIBS];
   VkDeviceSize buffer_offsets[PIPE_MAX_ATTRIBS];
   const struct zink_vertex_elements_state *elems = ctx->element_state;
   for (unsigned i = 0; i < elems->hw_state.num_bindings; i++) {
      struct pipe_vertex_buffer *vb = ctx->buffers + ctx->element_state->binding_map[i];
      assert(vb);
      if (vb->buffer.resource) {
         struct zink_resource *res = zink_resource(vb->buffer.resource);
         buffers[i] = res->buffer;
         buffer_offsets[i] = vb->buffer_offset;
         zink_resource_buffer_barrier(batch->cmdbuf, res, VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT,
                                      VK_PIPELINE_STAGE_VERTEX_INPUT_BIT);
         zink_batch_reference_resource_rw(batch, res, false);
      } else {
         buffers[i] = zink_resource(ctx->dummy_buffer)->buffer;
         buffer_offsets[i] = 0;
      }
   }

   if (elems->hw_state.num_bindings > 0)
      vkCmdBindVertexBuffers(batch->cmdbuf, 0,
                             elems->hw_state.num_bindings,
                             buffers, buffer_offsets);
}
```
There's now a barrier being added through the `zink_resource_buffer_barrier()` helper function, using the buffer's current `VkAccessFlags` and `VkPipelineStageFlags` from its last state when a change is needed. This ensures that the driver knows it's transitioning the buffer from its previous usage to the current usage, that of being a vertex input.

But wait, how does the pipeline know that the buffers have been used for xfb previously?

More changes are needed.

Currently, when zink binds xfb buffers again after pausing transform feedback (and only in this case), the following function will be called:
```c
static void
zink_emit_xfb_counter_barrier(struct zink_context *ctx)
{
   /* Between the pause and resume there needs to be a memory barrier for the counter buffers
    * with a source access of VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT
    * at pipeline stage VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT
    * to a destination access of VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT
    * at pipeline stage VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT.
    *
    * - from VK_EXT_transform_feedback spec
    */
   VkBufferMemoryBarrier barriers[PIPE_MAX_SO_OUTPUTS] = {};
   unsigned barrier_count = 0;

   for (unsigned i = 0; i < ctx->num_so_targets; i++) {
      struct zink_so_target *t = zink_so_target(ctx->so_targets[i]);
      if (t->counter_buffer_valid) {
          barriers[i].sType = VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER;
          barriers[i].srcAccessMask = VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT;
          barriers[i].dstAccessMask = VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT;
          barriers[i].buffer = zink_resource(t->counter_buffer)->buffer;
          barriers[i].size = VK_WHOLE_SIZE;
          barrier_count++;
      }
   }
   struct zink_batch *batch = zink_batch_no_rp(ctx);
   vkCmdPipelineBarrier(batch->cmdbuf,
      VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT,
      VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT,
      0,
      0, NULL,
      barrier_count, barriers,
      0, NULL
   );
   ctx->xfb_barrier = false;
}
```
This works great for resuming xfb, but it doesn't particularly help with any other cases, such as using a buffer for stream output for the first time. Instead, an improvement would be to **check the buffer's pipeline states upon being set as a stream output target (`zink_set_stream_output_targets()`)** and then flag the need for barriers. At that point, the above function can be changed:
```c
static void
zink_emit_xfb_counter_barrier(struct zink_context *ctx)
{
   /* Between the pause and resume there needs to be a memory barrier for the counter buffers
    * with a source access of VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT
    * at pipeline stage VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT
    * to a destination access of VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT
    * at pipeline stage VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT.
    *
    * - from VK_EXT_transform_feedback spec
    */
   struct zink_batch *batch = zink_batch_no_rp(ctx);
   for (unsigned i = 0; i < ctx->num_so_targets; i++) {
      struct zink_so_target *t = zink_so_target(ctx->so_targets[i]);
      struct zink_resource *res = zink_resource(t->counter_buffer);
      if (t->counter_buffer_valid)
          zink_resource_buffer_barrier(batch->cmdbuf, res,
                                       VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT,
                                       VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT);
      else
          zink_resource_buffer_barrier(batch->cmdbuf, res,
                                       VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT,
                                       VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT);
   }
   ctx->xfb_barrier = false;
}
```
Now the barriers are set as needed upon resuming xfb, but barriers are also emitted upon beginning xfb, which ensures synchronization both for that starting-xfb point as well as any future usage of the buffer, like as a vertex input.

I've already done some work to add similar changes throughout the driver, which should yield improved memory consistency for buffers as well as performance gains when a new renderpass would otherwise need to be started after an unnecessary queue submission.
