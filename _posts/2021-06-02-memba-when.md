---
published: false
---
## Remember When...

I said I'd be blogging every day about some changes? And that was a month ago or however long it's been? And we all had a good chuckle at the idea that I could blog every day like how things used to be?

Yeah, I remember that too.

Anyway, Bas still hasn't blogged, so let's check the blogenda:
* ~~handwaving about C++ draw templates~~
* ~~some obscure vbuf thing~~
* ~~shower~~
* make zink usable for gaming
* ~~complain about construction~~
* ~~improve shader caching~~
* this week's queue rewrite
* some other stuff
* suballocator?

I guess it's that time of the week again because the schedule says it's time to talk about this week's (or whenever it was) major rewrite of zink's queue handling. But first, only 90s kids will remember [that time](https://www.supergoodcode.com/architecture/) I blogged about a major queue rewrite and was excited to almost be hitting 70% of native performance.

## Synchronization
A common use of GL for big games is using multiple GL contexts to parallelize work. There's a lot of tricky restrictions for this, both on the app side and the driver side, but this is sort of the closest thing to multiple cmdbufs that GL provides.

We all recall how zink now uses a monotonic queue: upon commencing recording, each cmdbuf gets tagged with a 32bit integer id that doubles as a timeline semaphore id for fencing. The queue iterates, the cmdbuf counter increments, queue submission is done per-context in a thread, the GPU gets triangles, everyone is happy.

But how well does that work out with multiple contexts?

Pretty well, it turns out, as long as you're using a Vulkan driver that doesn't actually check to ensure you're using monotonic ids for your timeline values. Let's check out a totally hypothetical scenario that isn't just Steam:

* Have contexts A and B
* Context A starts recording, gets id 1 (nonzero id club represent)
* Context B starts recording, gets id 2
* Context A finishes recording, submits cmdbuf
* Context B finishes recording, submits cmdbuf
* Timeline wait on id 1
* Timeline wait on id 2

So far so good. But then we get past the "Checking for updates" window:
* Context A starts recording, gets id 3
* Context B starts recording, gets id 4
* Context B finishes recording, submits cmdbuf
* Context A finishes recording, submits cmdbuf
* Timeline wait on id 3
* Timeline wait on id 4

[![thonking.png]({{site.url}}/assets/thonking.png)]({{site.url}}/assets/thonking.png)

So now context B's submit thread is dumping cmdbuf 4's triangles into the GPU, then context A's submit thread is also trying to dump cmdbuf 3's triangles into the GPU, but the wait order for the timeline is still A -> B, meaning that the values are not monotonic.

**Will any drivers care?**

Magic 8-ball says no, no drivers care about this and everything still works fine. That's cool and interesting, but probably it'd be better to not do that.

The problem here is two problems:
* the queue submission thread is context-based when it should be screen based
* cmdbufs get an id when they start recording, not when they get submitted

The first problem is easy to fix: just deduplicate the thread and move the struct member.

The second one is trickier because everything in zink relies on cmdbufs getting an id as soon as they become active. This is done so that any resources written to by a given cmdbuf can have their usage tracked for synchronization purposes, e.g., reading back a buffer only after all its writes have landed.

The problem is further complicated by zink not having a great API barrier between directly accessing the "usage" for a resource and the value itself, by which I mean parts of the codebase directly reading the integer value vs having an API to wrap it; the latter would enable replacing the mechanism with whatever I wanted, so I decided to start by creating such a wrapper:

```c
struct zink_batch_usage {
   uint32_t usage;
   bool unflushed;
};
```

This is the existing struct but now with a bool value indicating that this cmdbuf is still recording. Each cmdbuf batch now has this sub-struct inlined onto it, and resources in zink can take references (pointers) to a specific cmdbuf's usage struct. Because batches are never destroyed, this means the wrapper API can always dereference the struct to determine how to synchronize the usage: if it's unflushed, it can flush or sync the flush thread; if it's real, pending usage, it can safely wait on that usage as a timeline value and guarantee monotonic ordering.