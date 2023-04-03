---
published: false
---
## Release Work

As everyone is well aware, the Mesa 23.1 branchpoint is scheduled for next week.

As everyone is also well aware, this is the release in which I've made promises about the viability of gaming on Zink.

Specifically, it now be viable**\***.

But exactly one**\*** [bug](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8699) [remains](https://gitlab.freedesktop.org/mesa/mesa/-/issues/6024) [as](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8746) [a](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8257) [blocker](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8328) to that. Just one.

So naturally I had to fix it quick before anyone noticed.

## The Bug: Background
The thing about OpenGL games is a lot of them are x86 binaries, which means they run in a 32bit process. Any 32bit application gets 32bit address space. 32bit address space means a 4GiB limit on addressable memory. But what does that mean?

**What is addressable memory?** Addressable memory is any memory that can be accessed by a process. If `malloc` is called, this memory is addressable. If a file is `mmap`ed, this memory is addressable. If GPU memory is mapped, this memory is addressable.

**What happens if the limit is exceeded?** Boom.

**Why is the limit only 4GiB?** Stop asking hard questions.

**Why is this difficult?** The issue from a driver perspective is that this limit includes both the addressable memory from the game (e.g., the game's internal `malloc` calls) as well as the addressable memory from the driver (e.g., all the GPU mapped memory). Thus, while I would like to have all 4GiB (or more, really; doesn't everyone have 256GiB RAM in their device these days?) to use with Zink, I do not have that luxury.

## The Bug: How Bad Is It?
Judging by recent bug reports and the prevalance on 32bit games, it's pretty bad. So let's check out some profiling.

