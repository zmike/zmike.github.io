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

Yes, currently some changes I've done allow this test to complete in **16% of the time** that it requires in the released version of Mesa. How did this happen?

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

Now here's the current `struct zink_context::fence_finish` hook for waiting on a fence:
```c
bool
zink_fence_finish(struct zink_screen *screen, struct zink_fence *fence,
                  uint64_t timeout_ns)
{
   bool success = vkWaitForFences(screen->dev, 1, &fence->fence, VK_TRUE,
                                  timeout_ns) == VK_SUCCESS;
   if (success && fence->active_queries)
      zink_prune_queries(screen, fence);
   return success;
}
```
Not much to see here. I've made some additions:
```c
static inline void
fence_remove_resource_access(struct zink_fence *fence, struct zink_resource *res)
{
   p_atomic_set(&res->batch_uses[fence->batch_id], 0);
}

bool
zink_fence_finish(struct zink_screen *screen, struct zink_fence *fence,
                  uint64_t timeout_ns)
{
   bool success = vkWaitForFences(screen->dev, 1, &fence->fence, VK_TRUE,
                                  timeout_ns) == VK_SUCCESS;
   if (success && fence->active_queries)
      zink_prune_queries(screen, fence);
   if (success) {
      if (fence->active_queries)
         zink_prune_queries(screen, fence);

      /* unref all used resources */
      util_dynarray_foreach(&fence->resources, struct pipe_resource*, pres) {
         struct zink_resource *res = zink_resource(*pres);
         fence_remove_resource_access(fence, res);

         pipe_resource_reference(pres, NULL);
      }
      util_dynarray_clear(&fence->resources);
   }
   return success;
}
```
Now as soon as a fence completes, all used resources will have the usage for the corresponding batch removed.

With this done, some changes can be made to `zink_transfer_map()`:
```c
static uint32_t
get_resource_usage(struct zink_resource *res)
{
   uint32_t batch_uses = 0;
   for (unsigned i = 0; i < 4; i++) {
      uint8_t used = p_atomic_read(&res->batch_uses[i]);
      if (used & ZINK_RESOURCE_ACCESS_READ)
         batch_uses |= ZINK_RESOURCE_ACCESS_READ << i;
      if (used & ZINK_RESOURCE_ACCESS_WRITE)
         batch_uses |= ZINK_RESOURCE_ACCESS_WRITE << i;
   }
   return batch_uses;
}

static void *
zink_transfer_map(struct pipe_context *pctx,
                  struct pipe_resource *pres,
                  unsigned level,
                  unsigned usage,
                  const struct pipe_box *box,
                  struct pipe_transfer **transfer)
{
   ...
   uint32_t batch_uses = get_resource_usage(res);
   if (pres->target == PIPE_BUFFER) {
      if ((usage & PIPE_TRANSFER_READ && batch_uses >= ZINK_RESOURCE_ACCESS_WRITE) ||
          (usage & PIPE_TRANSFER_WRITE && batch_uses)) {
         /* need to wait for rendering to finish
          * TODO: optimize/fix this to be much less obtrusive
          * mesa/mesa#2966
          */
         zink_fence_wait(pctx);
      }
```
`get_resource_usage()` iterates over `struct zink_resource::batch_uses` to generate a full bitmask of the resource's usage across all batches. Then, using the usage detail from the transfer_map, this can be applied in order to determine whether any waiting is necessary:
* If the transfer_map is for reading, only wait for rendering if the resource has pending writes
* If the transfer_map is for writing, wait if the resource has any pending usage

When this resource usage flagging is properly applied to all types of resources, huge performance gains abound even despite how simple it is. For the above test case, flagging the sampler view resources with read-only access continues to ensure their lifetimes while enabling them to be concurrently read from while draw commands are still pending, which yields the improvements to the `tex3d-npot` test case above.

## Future Gains
I'm always on the lookout for some gains, so I've already done and flagged some future work to be done here as well:
* texture resources have had similar mechanisms applied for improved synchronization, whereas currently they have none
* `zink-wip` has facilities for waiting on specific batches to complete, so `zink_fence_wait()` here can instead be changed to only wait on the last batch with write access for the `PIPE_TRANSFER_READ` case
* buffer ranges can be added to `struct zink_resource` as many other drivers use with the `util_range` API, enabling the fencing here to be skipped if the regions have no overlap or no valid data is extant/pending

These improvements are mostly specific to unit testing, but for me personally, those are the most important gains to be making, as faster test runs mean faster verification that no regressions have occurred, which means I can continue smashing more code into the repo.

Stay tuned for future Zink Gains updates.