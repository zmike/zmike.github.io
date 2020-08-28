---
published: false
---
## A Different Friday

Taking a break from talking about all this crazy feature nonsense, let's get into some optimization. Specifically, why have some of these piglit tests been taking so damn long to run?

A great example of this is `spec@!opengl 2.0@tex3d-npot`.

Mesa 20.3:
`MESA_LOADER_DRIVER_OVERRIDE=zink bin/tex3d-npot   24.65s user 83.38s system 73% cpu 2:27.31 total`

Mesa `zmike/zink-wip`:
`MESA_LOADER_DRIVER_OVERRIDE=zink bin/tex3d-npot   6.09s user 5.07s system 48% cpu 23.122 total`

Yes, currently some changes I've done allow this test to complete in 16% of the time that it requires in the released version of Mesa. How did this happen?

## Speed Loop
The core of the problem at present is zink's reliance on explicit fencing without enough info to know when to actually wait on a fence, as is vaguely referenced in [this ticket](https://gitlab.freedesktop.org/mesa/mesa/-/issues/2966), though the specific test case there still has yet to be addressed. In short, here's the problem codepath that's being hit for the above test:
```c
static void *
zink_transfer_map(struct pipe_context *pctx,
                  struct pipe_resource *pres,
                  unsigned level,
                  unsigned usage,
                  const struct pipe_box *box,
                  struct pipe_transfer **transfer)
{
   struct zink_context *ctx = zink_context(pctx);
   struct zink_screen *screen = zink_screen(pctx->screen);
   struct zink_resource *res = zink_resource(pres);

   struct zink_transfer *trans = slab_alloc(&ctx->transfer_pool);
   if (!trans)
      return NULL;

   memset(trans, 0, sizeof(*trans));
   pipe_resource_reference(&trans->base.resource, pres);

   trans->base.resource = pres;
   trans->base.level = level;
   trans->base.usage = usage;
   trans->base.box = *box;

   void *ptr;
   if (pres->target == PIPE_BUFFER) {
      if (usage & PIPE_TRANSFER_READ) {
         /* need to wait for rendering to finish
          * TODO: optimize/fix this to be much less obtrusive
          * mesa/mesa#2966
          */
         struct pipe_fence_handle *fence = NULL;
         pctx->flush(pctx, &fence, PIPE_FLUSH_HINT_FINISH);
         if (fence) {
            pctx->screen->fence_finish(pctx->screen, NULL, fence,
                                       PIPE_TIMEOUT_INFINITE);
            pctx->screen->fence_reference(pctx->screen, &fence, NULL);
         }
      }
```
Here, the current command buffer is submitted and then its fence is waited upon any time a call to e.g., `glReadPixels()` is made.

Any time.

Regardless of whether the resource in question even has pending writes.

Or whether it's ever had anything written to it at all.

This patch was added at my request to fix up a huge number of test cases we were seeing that failed due to cases of write -> read on a texture without waiting for the write to complete, and it was added with the understanding that at some time in the future it would be improved to both ensure synchronization and not incur such a massive performance hit.

That time has passed, and we are now in the future.

## Buffer Synchronization
At its core, this is just another synchronization issue, and so by adding more details about synchronization needs, the problem can be resolved.

I chose to do this using r/w flags for resources based on the command buffer (batch) id that the resource was used on. Presently, Mesa releases ship with this code for tracking resource usage in a command buffer:
```c
void
zink_batch_reference_resoure(struct zink_batch *batch,
                             struct zink_resource *res)
{
   struct set_entry *entry = _mesa_set_search(batch->resources, res);
   if (!entry) {
      entry = _mesa_set_add(batch->resources, res);
      pipe_reference(NULL, &res->base.reference);
   }
}
```
This ensures that the resource isn't destroyed before the batch finishes, which is a key functionality that drivers generally prefer to have in order to avoid crashing.

It doesn't, however, provide any details about how the resource is being used, such as whether it's being read from or written to, which means there's no way to optimize that case in `zink_transfer_map()`.

Here's the somewhat improved version:
```c
void
zink_batch_reference_resource_rw(struct zink_batch *batch, struct zink_resource *res, bool write)
{
   unsigned mask = write ? ZINK_RESOURCE_ACCESS_WRITE : ZINK_RESOURCE_ACCESS_READ;

   struct set_entry *entry = _mesa_set_search(batch->resources, res);
   if (!entry) {
      entry = _mesa_set_add(batch->resources, res);
      pipe_reference(NULL, &res->base.reference);
   }
   /* the batch_uses value for this batch is guaranteed to not be in use now because
    * reset_batch() waits on the fence and removes access before resetting
    */
   res->batch_uses[batch->batch_id] |= mask;
}
```
For context, I've simultaneously added a member `uint8_t batch_uses[4];` to `struct zink_resource`, as there are 4 batches in the batch array.

What this change does is allow callers to provide very basic info about whether the resource is being read from or written to in a given batch, stored as a bitmask to the batch-specific `struct zink_resource::batch_uses` member. Each batch has its own member of this array, as it needs to be able to be modified atomically in order to have its usage unset when a fence is notified of completion, and this member can have up to two bits.