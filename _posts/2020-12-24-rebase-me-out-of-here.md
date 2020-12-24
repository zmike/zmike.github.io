---
published: true
---
## ETOOMUCHREBASE

For real though, I've spent literal hours over the past week just rebasing stuff and managing conflicts. And then rebasing again after diffing against a reference commit when I inevitably discover that I fucked up the merge somehow.

But now the rebasing is done for a few minutes while I run more unit tests, so it's finally time to blog.

It's been a busy week. Nothing I've done has been very interesting. Lots of stabilizing and refactoring.

The blogging must continue, however, so here goes.

## The Return of QBOs
Many months ago I blogged about QBOs.

Maybe.

Maybe I didn't.

QBOs are are Query Buffer Objects, where the result of a given query is stored into a buffer. This is great for performance since it doesn't require any stalling while the query result is directly read.

Conceptually, anyway.

At present, zink has problems making this efficient for many types of queries due to the mismatch between GL query data and Vulkan query data, and there's a need to manually read it back and parse it with the CPU.

This is consistent with how zink manages non-QBO queries:
* start query
* end query
* stall GPU
* read query results back for user

As I've said many times along the way, the goal for zink has been to get the features in place and working first and then optimize later.

It's now later, and query bottlenecking is actually hurting performance in some apps (e.g., RPCS3).

## Memory++
Some profiling was done recently by bleeding edge tester Witold Baryluk, and it turns out that zink is using slightly less GPU memory than some native drivers, though it's also using slightly more than some other native drivers:

[![lowmem.png](https://i.imgur.com/7Umge5u.png)](https://i.imgur.com/7Umge5u.png)

Looking at the right side of the graph, it's obvious that there's still some VRAM available to be used, which means there's some VRAM available to use in optimizations.

As such, I decided to rewrite the query internals to have every query be a QBO internally, consistent with the RadeonSI model. While it does use a tiny bit more VRAM due to needing to allocate the backing buffers, the benefit of this is that now all query result data is copied to a buffer as soon as the query stops, so from an API perspective, this means that the result becomes available as soon as the backing buffer becomes idle.

It also means that any time actual QBOs are used (which is all the time for competent apps), I'll eventually have the ability to asynchronously post the result data from a query onto a user buffer without triggering a stall.

Functionally, this isn't a super complex maneuver: I've already got a utility function that performs a [vkCmdCopyQueryPoolResults](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdCopyQueryPoolResults.html) for regular QBO handling, so repurposing this to be called any time a query was ended, combined with modifying the parsing function to first map the internal buffer, was sufficient.

In the end, the query code is now a bit more uniform, and in the future I can use a compute shader to keep everything on GPU without needing to do any manual readback.
