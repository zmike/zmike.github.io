---
published: false
---
## I'm Trying

Blogging is tough, but I'm getting the posts out there one way or another.

Today marks what is likely to be the last of the "big" changes to zink in Mesa 21.1 before the merge window closes in less than two weeks, and what changes they were.

Threaded context support is now implemented, which, on its own, makes zink vaguely competitive against native GL drivers. I'd expect that for many scenarios, people should start seeing upwards of 60-70% native perf when previously the numbers were much lower, excepting things like furmark, where a weird problem with alpha blending is still causing a massive perf hit.

If that wasn't enough, my timeline semaphore handling also snuck in, providing a reduction in CPU overhead for asynchronous queue-related operations where supported.

