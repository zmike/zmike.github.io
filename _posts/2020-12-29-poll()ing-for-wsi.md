---
title: poll()ing For WSI
published: true
---
## A New Sync

For some time now I've been talking about zink's [lack of WSI](https://gitlab.freedesktop.org/mesa/mesa/-/issues/3608) and the forthcoming, near-messianic work by FPS sherpa Adam Jackson to [implement it](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/7661).

This is an extremely challenging project, however, and some work needs to be done in the meanwhile to ensure that zink doesn't drive off a cliff.

## Swapchain Strikes Back
Any swapchain master is already well acquainted with the mechanism by which images are displayed on the screen, but the gist of it for anyone unfamiliar is that there's N image resources that are swapped back and forth (2 for double-buffered, 3 for triple-buffered, ...). An image being rendered to is a *backbuffer*, and an image being displayed is a *frontbuffer*.

Ideally, a frontbuffer shouldn't be drawn to while it's in the process of being presented since such an action obliterates the app's usefulness. The knowledge of exactly when a resource is done presenting is gained through WSI. On Xorg, however, it's a bit tricky, to say the least. DRI3 is intended to address the underlying problems there with the XPresent extension, and the Mesa DRI frontend utilizes this to determine when an image is safe to use.

All this is great, and I'm sure it works terrifically in other cases, but zink is not like other cases. Zink lacks WSI integration. Under Xorg, this means it relies entirely on the DRI frontend to determine when it's safe to start rendering onto an image resource.

But what if the DRI frontend gets it wrong?

Indeed, due to quirks in the protocol/xserver, XPresent idle events can be received for a "presented" image immediately, even if it's still in use and has not finished presenting.

[![scumbag-xorg.png]({{site.url}}/assets/scumbag-xorg.png)]({{site.url}}/assets/scumbag-xorg.png)

In apps like SuperTuxKart, this results in insane flickering sure to be unsafe for anyone with epillepsy due to always rendering over the current frame before it's finished being presented.

## Return Of The Poll
To solve this problem, a wise, reclusive ghostwriter took time off from being at his local pub to offer me a suggestion:

Why not just rip the implicit fence of the DMAbuf out of the image object?

It was a great idea. But what did this pub enthusiast mean?

In short, WSI handles this problem by internally `poll()`ing on the image resource's underlying file descriptor. When there's no more events to `poll()` for, the image is safe to write on.

So now it's back to the (semi) basics of programming. First, get the file descriptor of the image using normal Vulkan function calls:
```c
static int
get_resource_fd(struct zink_screen *screen, struct zink_resource *res)
{
   VkMemoryGetFdInfoKHR fd_info = {};
   int fd;
   fd_info.sType = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR;
   fd_info.memory = res->obj->mem;
   fd_info.handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT;
   VkResult result = (*screen->vk_GetMemoryFdKHR)(screen->dev, &fd_info, &fd);
   return result == VK_SUCCESS ? fd : -1;
}
```
This provides a file descriptor that can be used for more nefarious purposes. Any time the gallium `pipe_context::flush` hook is called, the flushed resource (swapchain image) must be synchronized by `poll()`ing as in this snippet:
```c
static void
zink_flush(struct pipe_context *pctx,
           struct pipe_fence_handle **pfence,
           enum pipe_flush_flags flags)
{
   struct zink_context *ctx = zink_context(pctx);

   if (flags & PIPE_FLUSH_END_OF_FRAME && ctx->flush_res) {
      if (ctx->flush_res->obj->fd != -1) {
          /* FIXME: remove this garbage once we get wsi */
          struct pollfd p = {};
          p.fd = ctx->flush_res->obj->fd;
          p.events = POLLOUT;
          assert(poll(&p, 1, -1) == 1);
          assert(p.revents & POLLOUT);
      }
      ctx->flush_res = NULL;
   }
```
The `POLLOUT` event flag is used to determine [when it's safe to write](https://linux.die.net/man/2/poll). If there's no pending usage during present then this will return immediately, otherwise it will wait until the image is safe to use.

hacks++.
