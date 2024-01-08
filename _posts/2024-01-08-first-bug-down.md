---
published: false
---
# Slow Start

It's been a slow start to the year, by which I mean I've been buried under an absolute deluge of all the things you can imagine and then also a blizzard. The literal kind, not the kind that used to make great games.

Anyway, it's not all fun and specs in my capacity as CEO of OpenGL. Sometimes I gotta do Real Work. The number one source of Real Work, as always, is ~~my old code~~ the mesa bug tracker.

Unfortunately, the thing is completely overloaded with NVIDIA bugs right now, so it was slim pickins.

# Another Game I've Never Heard Of
Am I a boomer? Is this what being a boomer feels like? I really have lived long enough to see myself become the villain.

Next bug up is from this game called [Valheim](https://store.steampowered.com/app/892970/Valheim/). I think it's a LARPing chess game? Something like that? Don't @ me.

[This report](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10386) came in hot over the break with some rad new shading techniques:

[![hm](https://gitlab.freedesktop.org/mesa/mesa/uploads/549fc90c96a105272133823b090a4ba2/valheim-glitch-4.png)](https://gitlab.freedesktop.org/mesa/mesa/uploads/549fc90c96a105272133823b090a4ba2/valheim-glitch-4.png)

It looks way cooler if you play the trace, but you get the idea.

# Pinpoint Accuracy
First question: what in the Sam Hill is going on here?

Apparently `RADV_DEBUG=hang` fixes it, which was a curious one since no other env vars affected the issue. This means the problem is somehow caused by an issue related to the actual Vulkan queue submissions, since (according to legendary multipatch chef Samuel "[PLZ SEND REVIEWS!!](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/26930)" Pitoiset) this flag synchronizes the queue after every submit.

It's therefore no surprise that renderdoc was useless. When viewed in isolation, each frame is perfect, but when played at speed the synchronization is lost.

My first stops, as anyone would expect, were the sites of queue submission in zink. This means flush and present.

Now, I know not everyone is going to be comfortable taking this kind of wild, unhinged guess like I did, but stick with me here. The first thing I checked was a breakpoint on `zink_flush()`, which is where API flush calls filter through. There were the usual end-of-frame hits, but there were a fair number of calls originating from [glFenceSync](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glFenceSync.xhtml), which is the way a developer can subtly inform a GL driver that they definitely know what they're doing.

So I saw these calls coming in, and I stepped through `zink_flush()`, and I reached [this](https://gitlab.freedesktop.org/mesa/mesa/-/blob/b06f6e00fba6e33c28a198a1bb14b89e9dfbb4ae/src/gallium/drivers/zink/zink_context.c#L3866) spot:

```c
if (!batch->has_work) {
<-----HERE
      if (pfence) {
         /* reuse last fence */
         fence = ctx->last_fence;
      }
      if (!deferred) {
         struct zink_batch_state *last = zink_batch_state(ctx->last_fence);
         if (last) {
            sync_flush(ctx, last);
            if (last->is_device_lost)
               check_device_lost(ctx);
         }
      }
      if (ctx->tc && !ctx->track_renderpasses)
      tc_driver_internal_flush_notify(ctx->tc);
} else {
   fence = &batch->state->fence;
   submit_count = batch->state->usage.submit_count;
   if (deferred && !(flags & PIPE_FLUSH_FENCE_FD) && pfence)
      deferred_fence = true;
   else
      flush_batch(ctx, true);
}
```

Now this is a real puzzler, because if you know what you're doing as a developer, you shouldn't be reaching this spot. This is the penalty box where I put all the developers who *don't* know what they're doing, the spot where I push up my massive James Webb Space Telescope glasses and say, "No, ackchuahlly you don't want to flush right now." Because you only reach this spot if you trigger a flush when there's nothing to flush.

OR DO YOU?

For hahas, I noped out the first part of that conditional, ensuring that all flushes would translate to queue submits, and magically the bug went away. It was a miracle. Until I tried to think through what must be happening for that to have any effect.

# Synchronization: You Cannot Escape
The reason this was especially puzzling is the call sequence was:
* end-of-frame flush
* present
* glFenceSync flush

which means the last flush was optimized out, instead returning the fence from the end-of-frame flush. And these *should* be identical in terms of operations the app would want to wait on.

Except that there's a present in there, and technically that's a queue submission, and *technically* something might want to know if the submit for that has completed?

Why yes, that *is* stupid, but here at SGC, stupidity is our sustenance.

Anyway, I [blasted out](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/26935) a quick fix, and now you can all go play your favorite chess sim on your favorite driver again.