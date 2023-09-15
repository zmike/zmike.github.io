---
published: false
---
# Not Mine

But if you're reading, thanks for everything.

# Glamorous

I planned to blog about it a while ago, but then I didn't and news sites have since broken the news: Zink from Mesa main can run finally xservers.

Yes, it's true. For the first time ever, you can install Mesa (from git) and use zink (with environment variables) to run your entire system (unless you're [on Intel](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24700)).

But what was so challenging about getting this to work? The answer won't surprise you.

# WSI
Fans of the blog know that I'm no fan of WSI. If I had my way, GPUs would render to output buffers that we could peruse at our leisure using whatever methods we had at our disposal. Alas, few others share my worldview and so we all must suffer.

The root of all evil when it comes to computers is synchronization. This is triply so for anything GPU-related, and when all this "display server" chicanery is added in the evilness value becomes one of those numbers so large that numerologists are still researching naming possibilities. There are two types of synchronization used with WSI:
* implicit sync - "just fucking do it"
* explicit sync - "I'll tell you exactly when to do it"

From a user perspective, the former has less code to manage. The downside is that on the driver side things become more complex, as implicit sync is effectively layered atop explicit sync.

Another way of looking at it is:
* implicit sync - OpenGL
* explicit sync - Vulkan

And, since xservers run on GL, you can see where this is going.

# Implicitly Terrible
Don't get me wrong, explicit sync sucks too, but at least it makes sense. Broadly speaking, with explicit sync you have a dmabuf image, you submit it to the GPU, and you tell the server to display it.

In the words of noted Xorg developer, EGL maintainer, and synchronization PTSD survivor Daniel Stone, the way to handle implicit sync is "vibes". You have a dmabuf image, you `glFlush`, and magically it gets displayed.

Sound nuts? It is, and that's why Vulkan doesn't support it.

But zink uses Vulkan, so...

# Send Eyebleach
Explicit sync is based on two concepts:
* import
* export

A user of a dmabuf waits on an export operation before using it (i.e., a wait semaphore), then signals an import operation at the end of a cmdbuf submission. Vulkan WSI handles this under the hood for users. But there's no way to use Vulkan WSI with imported dmabufs, which means this all has to be copy/pasted around to work elsewhere.

In zink, all that happens in an xserver scenario is apps import/export dmabufs, sample/render them, and then do queue submission. To successfully copy/paste the WSI code and translate this into explicit sync for Vulkan, it's necessary to be a bit creative with driver mechanics. The gist of it is:
* when doing a queue import (from FOREIGN) for a dmabuf, create and queue an export (`DMA_BUF_IOCTL_EXPORT_SYNC_FILE`) semaphore to be waited on before the current cmdbuf
* when triggering a barrier on any exported dmabuf, queue an import (`DMA_BUF_IOCTL_IMPORT_SYNC_FILE`) semaphore to be signaled after the current cmdbuf
* at submit time, serialize all the wait semaphores onto a separate queue submission before the main cmdbuf
* at submit time, serialize all the signal semaphores onto a separate queue submission after the main cmdbuf
* pray

Big thanks to Faith "ARB_shader_image_load_store" Ekstrand for god-tier rubberducking when I was in the home stretch of this undertaking.

Anyway I expect to be absolutely buried in bug reports by next week from all the people testing this.