---
published: true
---
## To Begin...
Stream output is another name for `xfb` closer to the driver level. Inside mesa (and gallium), we'll commonly see different types related to `struct pipe_stream_output_*` which provide info about how the `xfb` info is output to its corresponding buffers.

Here's some blocks from the Zink implementation, referenced heavily from the original work by David Airlie:

```C
static struct pipe_stream_output_target *
zink_create_stream_output_target(struct pipe_context *pctx,
                                 struct pipe_resource *pres,
                                 unsigned buffer_offset,
                                 unsigned buffer_size)
{
   struct zink_so_target *t;
   t = CALLOC_STRUCT(zink_so_target);
   if (!t)
      return NULL;

   t->base.reference.count = 1;
   t->base.context = pctx;
   pipe_resource_reference(&t->base.buffer, pres);
   t->base.buffer_offset = buffer_offset;
   t->base.buffer_size = buffer_size;

   /* using PIPE_BIND_CUSTOM here lets us create a custom pipe buffer resource,
    * which allows us to differentiate and use VK_BUFFER_USAGE_TRANSFORM_FEEDBACK_COUNTER_BUFFER_BIT_EXT
    * as we must for this case
    */
   t->counter_buffer = pipe_buffer_create(pctx->screen, PIPE_BIND_STREAM_OUTPUT | PIPE_BIND_CUSTOM, PIPE_USAGE_DEFAULT,
4);
   if (!t->counter_buffer) {
      FREE(t);
      return NULL;
   }

   return &t->base;
}
```

Here we have the `create_stream_output_target` hook for our `struct pipe_context`, which is called any time we're initializing stream output. This takes the previously-created stream output buffer (more on buffer creation in a future post) along with offset and size parameters for the buffer. It's necessary to create a counter buffer here so that we can use it to correctly save and restore states if `xfb` is paused or resumed (glPauseTransformFeedback and glResumeTransformFeedback).


```C
static void
zink_stream_output_target_destroy(struct pipe_context *pctx,
                                  struct pipe_stream_output_target *psot)
{
   struct zink_so_target *t = (struct zink_so_target *)psot;
   pipe_resource_reference(&t->counter_buffer, NULL);
   pipe_resource_reference(&t->base.buffer, NULL);
   FREE(t);
}
```

A simple destructor hook for the previously-created object. `struct pipe_resource` buffers are refcounted, so we always need to make sure to properly unref them here; the first parameter of `pipe_resource_reference` can be thought of as the resource to unref, and the second is the resource to ref.


```C
static void
zink_set_stream_output_targets(struct pipe_context *pctx,
                               unsigned num_targets,
                               struct pipe_stream_output_target **targets,
                               const unsigned *offsets)
{
   struct zink_context *ctx = zink_context(pctx);

   if (num_targets == 0) {
      for (unsigned i = 0; i < ctx->num_so_targets; i++)
         pipe_so_target_reference(&ctx->so_targets[i], NULL);
      ctx->num_so_targets = 0;
   } else {
      for (unsigned i = 0; i < num_targets; i++)
         pipe_so_target_reference(&ctx->so_targets[i], targets[i]);
      for (unsigned i = num_targets; i < ctx->num_so_targets; i++)
         pipe_so_target_reference(&ctx->so_targets[i], NULL);
      ctx->num_so_targets = num_targets;

      /* emit memory barrier on next draw for synchronization */
      if (offsets[0] == (unsigned)-1)
         ctx->xfb_barrier = true;
      /* TODO: possibly avoid rebinding on resume if resuming from same buffers? */
      ctx->dirty_so_targets = true;
   }
}
```

This is the hook that gets called any time `xfb` is started or stopped, whether from `glBeginTransformFeedback` / `glEndTransformFeedback` or `glPauseTransformFeedback` / `glResumeTransformFeedback`. It's used to pass the active stream output buffers to the driver context in preparation for upcoming draw calls. We store and ref++ the buffers on activate, when `targets` is non-NULL, and then we ref-- and unset the buffers on deactivate, when `targets` is non-NULL. On `glResumeTransformFeedback`, `offsets` is `-1` (technically `UINT_MAX` from underflow).

According to Vulkan spec, we need to emit a different memory barrier for synchronization if we're resuming `xfb`, so a flag is set in that case. Ideally some work can be done here to optimize the case of pausing and resuming the same output buffer set to avoid needing to do additional synchronization later on in the draw, but for now, slow and steady wins the race.
