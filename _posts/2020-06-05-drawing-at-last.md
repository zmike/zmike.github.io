---
published: false
---
## Quick Runthrough
This `xfb` blog series has gone on for a while now, and it'd be great if it ended soon. Unfortunately, there's a lot of corner cases which are being found by piglit, and the work and fixing continue.

Today let's look at some of the drawing code for `xfb`, since that's probably not going to be changing much in the course of fixing those corner cases.

```c
static void
zink_emit_stream_output_targets(struct pipe_context *pctx)
{
   struct zink_context *ctx = zink_context(pctx);
   struct zink_screen *screen = zink_screen(pctx->screen);
   struct zink_batch *batch = zink_curr_batch(ctx);
   VkBuffer buffers[PIPE_MAX_SO_OUTPUTS];
   VkDeviceSize buffer_offsets[PIPE_MAX_SO_OUTPUTS];
   VkDeviceSize buffer_sizes[PIPE_MAX_SO_OUTPUTS];

   for (unsigned i = 0; i < ctx->num_so_targets; i++) {
      struct zink_so_target *t = (struct zink_so_target *)ctx->so_targets[i];
      buffers[i] = zink_resource(t->base.buffer)->buffer;
      buffer_offsets[i] = t->base.buffer_offset;
      buffer_sizes[i] = t->base.buffer_size;
   }

   screen->vk_CmdBindTransformFeedbackBuffersEXT(batch->cmdbuf, 0, ctx->num_so_targets,
                                                 buffers, buffer_offsets,
                                                 buffer_sizes);
   ctx->dirty_so_targets = false;
}
```

This is a function called from `zink_draw_vbo()`, which is the `struct pipe_context::draw_vbo` hook for drawing primitives. Here, the streamout target buffers are [bound in Vulkan](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBindTransformFeedbackBuffersEXT.html) in preparation for the upcoming draw, passing along related info into the command buffer.

```c
if (ctx->xfb_barrier) {
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
   batch = zink_batch_no_rp(ctx);
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
if (ctx->dirty_so_targets)
   zink_emit_stream_output_targets(pctx);
if (so_target && so_target->needs_barrier) {
   /* A pipeline barrier is required between using the buffers as
    * transform feedback buffers and vertex buffers to
    * ensure all writes to the transform feedback buffers are visible
    * when the data is read as vertex attributes.
    * The source access is VK_ACCESS_TRANSFORM_FEEDBACK_WRITE_BIT_EXT
    * and the destination access is VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT
    * for the pipeline stages VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT
    * and VK_PIPELINE_STAGE_VERTEX_INPUT_BIT respectively.
    *
    * - 20.3.1. Drawing Transform Feedback
    */
   VkBufferMemoryBarrier barriers[1] = {};
   if (so_target->counter_buffer_valid) {
       barriers[0].sType = VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER;
       barriers[0].srcAccessMask = VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT;
       barriers[0].dstAccessMask = VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT;
       barriers[0].buffer = zink_resource(so_target->base.buffer)->buffer;
       barriers[0].size = VK_WHOLE_SIZE;
   }
   batch = zink_batch_no_rp(ctx);
   zink_batch_reference_resoure(batch, zink_resource(so_target->base.buffer));
   vkCmdPipelineBarrier(batch->cmdbuf,
      VK_PIPELINE_STAGE_TRANSFORM_FEEDBACK_BIT_EXT,
      VK_PIPELINE_STAGE_VERTEX_INPUT_BIT,
      0,
      0, NULL,
      ARRAY_SIZE(barriers), barriers,
      0, NULL
   );
   so_target->needs_barrier = false;
}
```

This is a block added to `zink_draw_vbo()` for synchronization of the `xfb` buffers. The counter buffer needs a specific type of barrier according to the spec, and the streamout target buffer needs a different type of barrier. These need to be emitted outside of a render pass, so `zink_batch_no_rp()` is used to get a batch that isn't currently in a render pass (ending the active batch if necessary to switch to a new one). Without these, vk-layers will output tons of errors and also probably your stream output will be broken.

```c
   if (ctx->num_so_targets) {
      for (unsigned i = 0; i < ctx->num_so_targets; i++) {
         struct zink_so_target *t = zink_so_target(ctx->so_targets[i]);
         if (t->counter_buffer_valid) {
            zink_batch_reference_resoure(batch, zink_resource(t->counter_buffer));
            counter_buffers[i] = zink_resource(t->counter_buffer)->buffer;
            counter_buffer_offsets[i] = t->counter_buffer_offset;
         } else
            counter_buffers[i] = NULL;
         t->needs_barrier = true;
      }
      screen->vk_CmdBeginTransformFeedbackEXT(batch->cmdbuf, 0, ctx->num_so_targets, counter_buffers, counter_buffer_offsets);
   }

/* existing code */
   if (dinfo->index_size > 0) {
      assert(dinfo->index_size != 1);
      VkIndexType index_type = dinfo->index_size == 2 ? VK_INDEX_TYPE_UINT16 : VK_INDEX_TYPE_UINT32;
      struct zink_resource *res = zink_resource(index_buffer);
      vkCmdBindIndexBuffer(batch->cmdbuf, res->buffer, index_offset, index_type);
      zink_batch_reference_resoure(batch, res);
      vkCmdDrawIndexed(batch->cmdbuf,
         dinfo->count, dinfo->instance_count,
         dinfo->start, dinfo->index_bias, dinfo->start_instance);
   } else {
/* new code */
      if (so_target && screen->tf_props.transformFeedbackDraw) {
         zink_batch_reference_resoure(batch, zink_resource(so_target->counter_buffer));
         screen->vk_CmdDrawIndirectByteCountEXT(batch->cmdbuf, dinfo->instance_count, dinfo->start_instance,
                                       zink_resource(so_target->counter_buffer)->buffer, so_target->counter_buffer_offset, 0,
                                       MIN2(so_target->stride, screen->tf_props.maxTransformFeedbackBufferDataStride));
      }
      else
         vkCmdDraw(batch->cmdbuf, dinfo->count, dinfo->instance_count, dinfo->start, dinfo->start_instance);
   }

   if (dinfo->index_size > 0 && dinfo->has_user_indices)
      pipe_resource_reference(&index_buffer, NULL);

   if (ctx->num_so_targets) {
      for (unsigned i = 0; i < ctx->num_so_targets; i++) {
         struct zink_so_target *t = zink_so_target(ctx->so_targets[i]);
         counter_buffers[i] = zink_resource(t->counter_buffer)->buffer;
         counter_buffer_offsets[i] = t->counter_buffer_offset;
         t->counter_buffer_valid = true;
      }
      screen->vk_CmdEndTransformFeedbackEXT(batch->cmdbuf, 0, ctx->num_so_targets, counter_buffers, counter_buffer_offsets);
   }
```

Excluding a small block that I've added a comment for, this is pretty much all added for handling `xfb` draws. This includes the begin/end calls for `xfb` and outputting to the counter buffers for each streamout target, and the actual `vkCmdDrawIndirectByteCountEXT` call for drawing transform feedback when appropriate.

The begin/end calls handle managing the buffer states to work with `glPauseTransformFeedback` and `glResumeTransformFeedback`. When resuming, the counter buffer offset is used to track the state and continue with the buffers from the correct location in memory.

## Next time
We'll look at `xfb` queries and extension/feature enabling, and I'll start to get into a bit more detail about how more of this stuff works.