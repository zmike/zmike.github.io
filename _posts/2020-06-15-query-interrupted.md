---
published: false
---
## Short post today

I got caught up in some exciting synchronization code today and ran out of time to write a post before my brain started to shut down. As a result, let's talk briefly about synchronization.

## What is synchronization?
Synchronization is the idea that application and gpu states match up. When used properly, it means that when a vertex buffer is thrown into a shader, the buffer contains the expected data. When not used properly, nothing works at all.

## Zink and synchronization (zinkronization?)
Zink uses the concept of "batches", aka `struct zink_batch` to represent Vulkan command buffers. Zink throws a stream of commands into a batch, and eventually they get executed by the driver. A batch doesn't necessarily terminate with a draw command, and not every batch will even contain a draw command.

Batches in Zink are dispatched (flushed) to the gpu based on [render passes](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes). Some commands must be made inside a render pass, and some must be made outside. When that boundary is reached, and the next command must be inside/outside a render pass while the current batch is outside/inside a render pass, the current batch is flushed and the next batch in the circular batch queue becomes current.

However, just because a batch is flushed doesn't mean it has been executed.

## Fences
A fence is an object in Zink (and other places) which provides a notification when a batch has completed. In the case where, e.g., an application wants to read from an in-flight draw buffer, the current batch is flushed, then a fence is waited on to determine when the pending draw commands are complete. This is synchronization for command buffers.

## Barriers
Barriers are synchronization for memory access. They allow fine-grained control over exactly when a resource transitions between access states. This avoids data races between e.g., a write operation and a read operation which might be occurring in the same command buffer, but it enables the user of the Vulkan API to be explicit about the usage to maximize performance.

## Query synchronization
A big issue with the existing query handling, as was previously discussed relates to synchronization. The queries need to be able to keep the underlying data that they're fetching from the driver in sync with the application's idea of what data the query is fetching. This needs to remain consistent across batch flushes and query pool overflows.

More on this to come.