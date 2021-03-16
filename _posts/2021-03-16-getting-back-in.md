---
published: false
---
## Superposition

I got a [weird bug report](https://gitlab.freedesktop.org/zmike/mesa/-/issues/71) the other day. Apparently Unigine Superposition, the final boss of Unigine benchmarks, was broken in zink(-wip). And it was a regression, which meant that at some point it had worked, but because testing is hard, it then stopped working for a while.

## The Problem (superposition wtf u doin?)
I had no idea what the problem was other than a vague impression it might be queue-related given the perf output in the bug report. The first step here was to figure out what I was dealing with. Off I went to the two horsemen of all queue-related bugs: `zink_flush()` and `zink_fence_finish()`.

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

In short, the control flow goes:
* if the GPU rejected a previous cmdbuf, fail
* detect and then submit deferred flushes (i.e., the app has requested a sync object and at some later point decided to wait on it)
* verify that the cmdbuf represented by this fence has actually been submitted
* detect cases where the internal fence (`mfence->fence`) no longer belongs to the gallium sync object (`mfence`)
* finally, attempt to actually wait on the fence

So this is all fine and good, and it's been working terrifically for many months. But I put a printf at the top, and it turns out this was being spammed nonstop during load with the app constantly checking the same sync object to see if it was finished.

It wasn't finished.

Specifically, the `return fence ? p_atomic_read(&fence->completed) : false;` case was being hit, which led me to dig deeper into what was going on.

Another printf in `zink_flush()` (function contents omitted to spare the eyes of anyone under the legal drinking age) revealed that the sync object reaching `zink_fence_finish` was, in fact, the second one created by the app, which was then being waited upon twenty-something flushes later.

So in this case, `mfence->deferred_id` was something like `2` and the current batch id in `ctx->curr_batch` was around `20`. The last-completed batch id was also around `20`.

## Revision
A slight modification here resolved the issue:

```c
if (pctx && mfence->deferred_ctx == pctx) {
   if (mfence->deferred_id == ctx->curr_batch) {
      zink_context(pctx)->batch.has_work = true;
      /* this must be the current batch */
      pctx->flush(pctx, NULL, !timeout_ns ? PIPE_FLUSH_ASYNC : 0);
      if (!timeout_ns)
         return false;
   }
   /* this batch is known to have finished */
   if (mfence->deferred_id <= screen->last_finished)
      return true;
}
```

This way, cases where an app is randomly waiting on something that has finished so far in the past that there's no longer any trace of it can just become a no-op success, and everything is happy.

Probably.

Testing is ongoing, but it seems like everything is happy now.