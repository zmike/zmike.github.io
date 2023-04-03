---
published: false
---
## Release Work

As everyone is well aware, the Mesa 23.1 branchpoint is definitely going to be next week, and there is zero chance that it could ever be delayed\*.

As everyone is also well aware, this is the release in which I've made unbreakable\* promises about the viability of gaming on Zink.

Specifically, it will now be viable**\***.

But exactly one**\*** [bug](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8699) [remains](https://gitlab.freedesktop.org/mesa/mesa/-/issues/6024) [as](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8746) [a](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8257) [blocker](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8328) to that. Just one.

So naturally I had to fix it quick before anyone noticed\*.

\* Don't @me for factual inconsistencies in any of the previous statements.

## The Bug: Background
The thing about OpenGL games is a lot of them are x86 binaries, which means they run in a 32bit process. Any 32bit application gets 32bit address space. 32bit address space means a 4GiB limit on addressable memory. But what does that mean?

**What is addressable memory?** Addressable memory is any memory that can be accessed by a process. If `malloc` is called, this memory is addressable. If a file is `mmap`ed, this memory is addressable. If GPU memory is mapped, this memory is addressable.

**What happens if the limit is exceeded?** Boom.

**Why is the limit only 4GiB?** Stop asking hard questions.

**Why is this difficult?** The issue from a driver perspective is that this limit includes both the addressable memory from the game (e.g., the game's internal `malloc` calls) as well as the addressable memory from the driver (e.g., all the GPU mapped memory). Thus, while I would like to have all 4GiB (or more, really; doesn't everyone have 32GiB RAM in their device these days?) to use with Zink, I do not have that luxury.

## The Bug: How Bad Is It?
Judging by recent bug reports and the prevalance on 32bit games, it's pretty bad. Given that I solved GPU map VA leaking a long time ago, the culprit must be memory utilization in the driver. Let's check out some profiling.

The process for this is simple: capture a (long) trace from a game and then run it through massif. [Sound familiar]({{site.url}}/oom/)?

The game in this case, of course, Tomb Raider (2013), the home of our glorious triangle princess. Starting a new game runs through a lot of intro cinematics and loads a lot of assets, and the memory usage is explosive. See what I did there? Yeah, jokes. On a Monday. Whew I need a vacation.

But this is where I started:

[![leak.png]({{site.url}}/assets/mem/leak.png)]({{site.url}}/assets/mem/leak.png)

2.4 GiB memory allocated by the driver. In a modern, 64bit process, where we can make full use of the 64GiB memory in the device, this is not a problem and we can pretend to be a web browser using this much for a single tab. But here, from an era when memory management was important and everyone didn't have 128GiB memory available, that's not going to fly.

## 🤔
Initial analysis yielded the following pattern:

```
n3: 112105776 0x5F88B73: nir_intrinsic_instr_create (nir.c:759)
n1: 47129360 0x5F96216: clone_intrinsic (nir_clone.c:358)
n1: 47129360 0x5F9692E: clone_instr (nir_clone.c:496)
n1: 47129360 0x5F96BB4: clone_block (nir_clone.c:563)
n2: 47129360 0x5F96DEE: clone_cf_list (nir_clone.c:617)
n1: 46441568 0x5F971CE: clone_function_impl (nir_clone.c:701)
n3: 46441568 0x5F974A4: nir_shader_clone (nir_clone.c:774)
n1: 28591984 0x67D9DE5: zink_shader_compile_separate (zink_compiler.c:3280)
n1: 28591984 0x69005F8: precompile_separate_shader_job (zink_program.c:2022)
n1: 28591984 0x57647B7: util_queue_thread_func (u_queue.c:309)
n1: 28591984 0x57CD7BC: impl_thrd_routine (threads_posix.c:67)
n1: 28591984 0x4DDB14C: start_thread (in /usr/lib64/libc.so.6)
n0: 28591984 0x4E5BBB3: clone (in /usr/lib64/libc.so.6)
```

Looking at the code, I found an obvious issue: when I [implemented precompile for separate shaders](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21197) a month or two ago, I had a teensie weensie little bug. Turns out when memory is allocated, it has to be freed or else it becomes unreachable.

This is commonly called a leak.

It wasn't caught before now because it *only* affects Tomb Raider and a handful of unit tests.

But I caught it, and it was [so minor](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/22175) that I already ("quietly") landed the fix without anyone noticing.

This sort of thing will be fixed when zink is rewritten in Rust\*.

## 🤔🤔
Actual bugs fixed, what does memory utilization look like now?

[![baseline.png]({{site.url}}/assets/mem/baseline.png)]({{site.url}}/assets/mem/baseline.png)

Down 300MiB to 2.1GiB. A 12.5% reduction. Not that exciting.

Certainly nothing that would warrant a SGC blog post.

My readers have standards.

