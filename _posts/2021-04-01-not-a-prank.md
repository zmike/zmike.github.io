---
published: false
---
## I'm Trying

Blogging is tough, but I'm getting the posts out there one way or another.

Today marks what is likely to be the last of the "big" changes to zink in Mesa 21.1 before the merge window closes in less than two weeks, and what changes they were.

**Threaded context** support is now implemented, which, on its own, makes zink vaguely competitive against native GL drivers. I'd expect that for many scenarios, people should start seeing upwards of 60-70% native perf when previously the numbers were much lower, excepting things like furmark, where a weird problem with alpha blending is still causing a massive perf hit.

If that wasn't enough, my **timeline semaphore handling** also snuck in, providing a reduction in CPU overhead for asynchronous queue-related operations where supported. Special thanks to Vulkan crash test dummy and uninitialized variable enthusiast Lionel Landwerlin for tripping and falling over basically every line of code in this implementation to help get it to the finish line for your consumption.

And if that still wasn't enough, my [RADV draw dispatch refactor](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8788) also landed yesterday, both paving the way for some totally secret future work of mine and also bringing a 3-4% reduction in CPU overhead for draws that will make your gaming *feel* faster now that you're aware of it but realistically won't have any discernible effect. Basically racing stripes.