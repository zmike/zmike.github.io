---
published: false
---
## Superposition

I got a [weird bug report](https://gitlab.freedesktop.org/zmike/mesa/-/issues/71) the other day. Apparently Unigine Superposition, the final boss of Unigine benchmarks, was broken in zink(-wip). And it was a regression, which meant that at some point it had worked, but because testing is hard, it then stopped working for a while.

## The Problem
I had no idea what the problem was other than a vague impression it might be queue-related given the perf output in the bug report. The first step here was to figure out what I was dealing with. Off I went to the sources of all queue-related bugs: `zink_flush()` and `zink_fence_finish()`.

Here's the latter of the two:
```c
static bool
zink_fence_finish(struct zink_screen *screen, struct pipe_context *pctx, struct zink_tc_fence *mfence,
                  uint64_t timeout_ns)
{
   pctx = threaded_context_unwrap_sync(pctx);
   struct zink_context *ctx = zink_context(pctx);

   if (screen->device_lost)
      return true;

   if (pctx && mfence->deferred_ctx == pctx && mfence->deferred_id == ctx->curr_batch) {
      zink_context(pctx)->batch.has_work = true;
      /* this must be the current batch */
      pctx->flush(pctx, NULL, !timeout_ns ? PIPE_FLUSH_ASYNC : 0);
      if (!timeout_ns)
         return false;
   }

   /* need to ensure the tc mfence has been flushed before we wait */
   bool tc_finish = tc_fence_finish(ctx, mfence, &timeout_ns);
   struct zink_fence *fence = mfence->fence;
   if (!tc_finish || (fence && !fence->submitted))
      return fence ? p_atomic_read(&fence->completed) : false;

   /* this was an invalid flush, just return completed */
   if (!mfence->fence)
      return true;
   /* if the zink fence has a different batch id then it must have completed and been recycled already */
   if (mfence->fence->batch_id != mfence->batch_id)
      return true;

   return zink_vkfence_wait(screen, fence, timeout_ns);
}
```