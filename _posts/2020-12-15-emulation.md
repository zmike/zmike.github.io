---
published: false
---
## In Many Forms

One of the people in the #zink IRC channel has been posing an interesting challenge for me in the form of trying to run every possible console emulator on my zink-wip branch.

This has raised a number of issues with various parts of the driver, so expect a number of posts on the topic.

## Threads
First up was the [citra](https://github.com/citra-emu/citra/) emulator for the 3DS. This is an interesting app for a number of reasons, the least of which is because it uses a ton of threads, including a separate one for GL, which put my own work to the test.

Suffice to say that my initial implementation of `u_threaded_context` needed some work.

One of the main premises of the threaded context is this idea of an asynchronous fence object. The threaded context will create these in a thread and provide them to the driver in the `pipe_context::flush` hook, but only in some cases; at other times, the fence object provided will just be a "regular" synchronous one.

The trick here is that the driver itself has a fence for managing synchronization, and the threaded context can create N number of its own fences to manage the driver's fence, all of which must potentially work when called in a random order and from either the "main" thread or the driver-specific thread.

There's too much code involved here to be providing any samples here, but I'll go over the basics of it just for posterity. Initially, I had implemented this entirely on the zink side such that each zink fence had references to all the tc fences in a chain, and fence-related resources were generally managed on the last fence in the chain. I had two separate object types for this: one for zink fences and one for tc fences. The former contained all the required vulkan-specific objects while the latter contained just enough info to work with tc.

This was sort of fine, and it worked for many things, the least of which was all my benchmarking.

The problem was that a desync could occur if one of the tc fences was destroyed sufficiently later than its zink fence, leading to an eventual crash. This was never triggered by unit tests nor basic app usage, but something like citra with its many threads managed to hit it consistently and quickly.

Thus began the day-long process of rewriting the tc implementation to a much-improved 2.0 version. The primary difference in this design model is that I worked a bit closer to the original RadeonSI implementation, having only a single externally-used fence object type for both gallium as well as tc and creating them for the zink fence object without any sort of cross-referencing. This meant that rather than having 1 zink fence with references to N tc fences, I now had N tc fences each with a reference to 1 zink fence.

This simplified the code a bit in other ways after the rewrite, as the gallium/tc fence objects were now entirely independent. The one small catch was that zink fences get recycled, meaning that in theory a gallium/tc fence could have a reference to a zink fence that it no longer was managing, but this was simple enough to avoid by gating all tc fence functionality on a comparison between its stored fence id and the id of the fence that it had a reference to. If they failed to match, the gallium/tc fence had already completed.

## 