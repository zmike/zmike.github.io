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
A common use of GL for big games is using multiple GL contexts to parallelize work. There's a lot of tricky restrictions for this, both on the app side and the driver side, but this is more or less the closest thing to multiple cmdbufs that GL provides.