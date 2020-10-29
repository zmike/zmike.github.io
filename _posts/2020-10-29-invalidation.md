---
published: false
---
## Buffering

I've got a lot of exciting stuff in the pipe now, but for today I'm just going to talk a bit about resource invalidation: what it is, when it happens, and why it's important.

Let's get started.

## What is invalidation?
Resource invalidation occurs when the backing buffer of a resource is wholly replaced. Consider the following scenario under zink:
* Have `struct A { VkBuffer buffer; };`
* User calls `glBufferData(target, size, data, usage)`, which stores data to `A.buffer`
* User calls `glBufferData(target, size, NULL, usage)`, which unsets the data from `A.buffer`

On a sane/competent driver, the second `glBufferData` call will trigger *invalidation*, which means that `A.buffer` will be replaced entirely, while `A` is still the driver resource used by Gallium to represent `target`.

## When does invalidation occur?
Resource invalidation can occur in a number of scenarios, but the most common is when unsetting a buffer's data, as in the above example. The other main case for it is replacing the data of a buffer that's in use for another operation. In such a case, the backing buffer can be replaced to avoid forcing a sync in the command stream which will stall the application's processing. There's some other cases for this as well, like `glInvalidateFramebuffer` and `glDiscardFramebufferEXT`, but the primary usage that I'm interested in is buffers.

## Why is invalidation important?
The main reason is performance. In the above scenario without invalidation, the second `glBufferData` call will *write* null to the whole buffer, which is going to be much more costly than just creating a new buffer.

## That's it
Now comes the slightly more interesting part: how does invalidation work in zink?

Currently, as of today's mainline zink codebase, we have `struct zink_resource` to represent a resource for either a buffer or an image. One `struct zink_resource` represents exactly one `VkBuffer` or `VkImage`, and there's some passable lifetime tracking that I've written to guarantee that these Vulkan objects persist through the various command buffers that they're associated with.

Each `struct zink_resource` is, as is the way of Gallium drivers, also a `struct pipe_resource`, which is tracked by Gallium. Because of this, `struct zink_resource` objects themselves cannot be invalidated in order to avoid breaking Gallium, and instead only the inner Vulkan objects themselves can be replaced.

For this, I created `struct zink_resource_object`, which is an object that stores only the data that directly relates to the Vulkan objects, leaving `struct zink_resource` to track the states of these objects. Their lifetimes are separate, with `struct zink_resource` being bound to the Gallium tracker and `struct zink_resource_object` persisting for either the lifetime of `struct zink_resource` or its command buffer usage—whichever is longer.

## Code
The code for this mechanism isn't super interesting since it's basically just moving some parts around. Where it gets interesting is the exact mechanics of invalidation and how `struct zink_resource_object` can be injected into an in-use resource, so let's dig into that a bit.

Here's what the `pipe_context::invalidate_resource` hook looks like:
```c
static void
zink_invalidate_resource(struct pipe_context *pctx, struct pipe_resource *pres)
{
   struct zink_context *ctx = zink_context(pctx);
   struct zink_resource *res = zink_resource(pres);
   struct zink_screen *screen = zink_screen(pctx->screen);

   if (pres->target != PIPE_BUFFER)
      return;
```
This only handles buffer resources, but extending it for images would likely be little to no extra work.
```c
   if (res->valid_buffer_range.start > res->valid_buffer_range.end)
      return;
```
Zink tracks the valid data segments of its buffers. This conditional is used to check for an uninitialized buffer, i.e., one which contains no valid data. If a buffer has no data, it's already invalidated, so there's nothing to be done here.
```c
   util_range_set_empty(&res->valid_buffer_range);
```
Invalidating means the buffer will no longer have any valid data, so the range tracking can be reset here.
```c
   if (!get_all_resource_usage(res))
      return;
```
If this resource isn't currently in use, unsetting the valid range is enough to invalidate it, so it can just be returned right away with no extra work.
```c
   struct zink_resource_object *old_obj = res->obj;
   struct zink_resource_object *new_obj = resource_object_create(screen, pres, NULL, NULL);
   if (!new_obj) {
      debug_printf("new backing resource alloc failed!");
      return;
   }
```
Here's the old internal buffer object as well as a new one, created using the existing buffer as a template so that it'll match.
```c
   res->obj = new_obj;
   res->access_stage = 0;
   res->access = 0;
```
`struct zink_resource` is just a state tracker for the `struct zink_resource_object` object, so upon invalidate, the states are unset since this is effectively a brand new buffer.
```c
   zink_resource_rebind(ctx, res);
```
This is the tricky part, and I'll go into more detail about it below.
```c
   zink_descriptor_set_refs_clear(&old_obj->desc_set_refs, old_obj);
```
If this resource was used in any cached descriptor sets, the references to those sets need to be invalidated so that the sets won't be reused.
```c
   zink_resource_object_reference(screen, &old_obj, NULL);
}
```
Finally, the old `struct zink_resource_object` is unrefed, which will ensure that it gets destroyed once its current command buffer has finished executing.

Simple enough, but what about that `zink_resource_rebind()` call? Like I said, that's where things get a little tricky, but because of how much time I spent on descriptor management, it ends up not being too bad.

This is what it looks like:

```c
void
zink_resource_rebind(struct zink_context *ctx, struct zink_resource *res)
{
   assert(res->base.target == PIPE_BUFFER);
```
Again, this mechanism is only handling buffer resource for now, and there's only one place in the driver that calls it, but it never hurts to be careful.
```c
   for (unsigned shader = 0; shader < PIPE_SHADER_TYPES; shader++) {
      if (!(res->bind_stages & BITFIELD64_BIT(shader)))
         continue;
      for (enum zink_descriptor_type type = 0; type < ZINK_DESCRIPTOR_TYPES; type++) {
         if (!(res->bind_history & BITFIELD64_BIT(type)))
            continue;
```
Something common to many Gallium drivers is this idea of "bind history", which is where a resource will have bitflags set when it's used for a certain type of binding. While other drivers have a lot more cases than zink does due to various factors, the only thing that needs to be checked for my purposes is the descriptor type (UBO, SSBO, sampler, shader image) across all the shader stages. If a given resource has the flags set here, this means it was at some point used as a descriptor of this type, so the current descriptor bindings need to be compared to see if there's a match.
```c
         uint32_t usage = zink_program_get_descriptor_usage(ctx, shader, type);
         while (usage) {
            const int i = u_bit_scan(&usage);
```
This is a handy mechanism that returns the current descriptor usage of a shader as a bitfield. So for example, if a vertex shader uses UBOs in slots 0, 1, and 3, `usage` will be 11, and the loop will process `i` as 0, 1, and 3.
```c
            struct zink_resource *cres = get_resource_for_descriptor(ctx, type, shader, i);
            if (res != cres)
               continue;
```
Now the slot of the descriptor type can be compared against the resource that's being re-bound. If this resource is the one that's currently bound to the specified slot of the specified descriptor type, then steps can be taken to perform additional operations necessary to successfully replace the backing storage for the resource, mimicking the same steps taken when initially binding the resource to the descriptor slot.
```c
            switch (type) {
            case ZINK_DESCRIPTOR_TYPE_SSBO: {
               struct pipe_shader_buffer *ssbo = &ctx->ssbos[shader][i];
               util_range_add(&res->base, &res->valid_buffer_range, ssbo->buffer_offset,
                              ssbo->buffer_offset + ssbo->buffer_size);
               break;
            }
```
For SSBO descriptors, the only change needed is to add valid range for the bound region as . This region is passed to the shader, so even if it's never written to, it might be, and so it can be considered a valid region.
```c
            case ZINK_DESCRIPTOR_TYPE_SAMPLER_VIEW: {
               struct zink_sampler_view *sampler_view = zink_sampler_view(ctx->sampler_views[shader][i]);
               zink_descriptor_set_refs_clear(&sampler_view->desc_set_refs, sampler_view);
               zink_buffer_view_reference(ctx, &sampler_view->buffer_view, NULL);
               sampler_view->buffer_view = get_buffer_view(ctx, res, sampler_view->base.format,
                                                           sampler_view->base.u.buf.offset, sampler_view->base.u.buf.size);
               break;
            }
```
Sampler descriptors require a new `VkBufferView` be created since the previous one is no longer valid. Again, the references for the existing bufferview need to be invalidated now since that descriptor set can no longer be reused from the cache, and then the new `VkBufferView` is set after unrefing the old one.
```c
            case ZINK_DESCRIPTOR_TYPE_IMAGE: {
               struct zink_image_view *image_view = &ctx->image_views[shader][i];
               zink_descriptor_set_refs_clear(&image_view->desc_set_refs, image_view);
               zink_buffer_view_reference(ctx, &image_view->buffer_view, NULL);
               image_view->buffer_view = get_buffer_view(ctx, res, image_view->base.format,
                                                         image_view->base.u.buf.offset, image_view->base.u.buf.size);
               util_range_add(&res->base, &res->valid_buffer_range, image_view->base.u.buf.offset,
                              image_view->base.u.buf.offset + image_view->base.u.buf.size);
               break;
            }
```
Images are nearly identical to the sampler case, the difference being that while samplers are read-only like UBOs (and therefore reach this point already having valid buffer ranges set), images are more like SSBOs and can be written to. Thus the valid range must be set here like in the SSBO case.
```c
            default:
               break;
```
Eagle-eyed readers will note that I've omitted a UBO case, and this is because there's nothing extra to be done there. UBOs will already have their valid range set and don't need a `VkBufferView`.
```c
            }

            invalidate_descriptor_state(ctx, shader, type);
```
Finally, the incremental decsriptor state hash for this shader stage and descriptor type is invalidated. It'll be recalculated normally upon the next draw or compute operation, so this is a quick zero-setting operation.
```c
         }
      }
   }
}
```

That's everything there is to know about the current state of resource invalidation in zink!