---
published: false
---
## By Now

...or in the very near future, [the ol' bumperino](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/8989) will have landed, putting zink at GL 4.5.

But that's boring, so let's check out something very slightly more interesting.

## Steam Games

What are they and how do they work?

I'm not going to answer these questions, but I am going to be looking into getting them working on zink.

To that end, as I hinted at [yesterday]({{site.url}}/new-order/), I began with Wolfenstein: The New Order, as chosen by Daniel Schuermann, the lucky winner of the What Steam Game Should Zink Use As Its Primary Test Case And Benchmark contest that was recently held.

Early tests of this game were unimpressive. That is to say I got an immediate crash. It turns out that having the GL compatibility context restricted to 3.0 is bad for getting AAA games running, so zink-wip now enables 4.6 compat contexts.

But then I was still getting a crash without any clear error message. Suddenly, I was back in 2004 trying to figure out how to debug wine apps.

Things are much simpler now, however. `PROTON_DUMP_DEBUG_COMMANDS` enables dumping scripts for debugging from steam, including one which attaches a debugger to the game. This solved the problem of getting a debugger in before the almost-immediate crash, but it didn't get me closer to a resolution.

The problem now is that I'd attached a debugger to the in-wine process, which is just a sandbox for the Windows API. What I actually wanted was to attach to the wine process itself so I could see what was going on in the driver.

`gdb --pid=$(pidof WolfNewOrder_x64.exe)` ended up what I needed, but this was complicated by the fact that I had to attach before the game crashed and without triggering the steam error reporter. So in the end, I had to attach using the proton script, then while it was paused, attach to the outer process for driver debugging. But then also I had to attach to the outer process after zink

After cluelessly asking around in the DXVK discord, @Herbert helpfully provided a gdb python