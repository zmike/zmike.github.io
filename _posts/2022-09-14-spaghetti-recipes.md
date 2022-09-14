---
published: false
---
## Homemade Spaghetti

As I've mentioned in a couple posts, I'm somewhat of a spaghetti expert. In my opinion, the longer and more plentiful the spaghetti is, the better.

So let's take a look at one of my favorite spaghetti recipes.

## Step 1: Read The Label
Today's spaghetti comes from my new favorite brand of spaghetti feed, [vkoverhead](https://github.com/zmike/vkoverhead). It's a simple brand, but it really gets the job done when it comes to growing great spaghetti. This particular spaghetti feed is `vkoverhead -test 0`, which is the most simple type. It grows the kind of spaghetti that everyone notices because it's a staple of all graphics diets.

If I check out the state of this spaghetti feed with RADV now, I see the following:
```
$ ./vkoverhead -test 0 -output-only -duration 3
28345
```

Thus, I can see that I'm getting 28.3 million draws/second. Not too bad. Let's check AMDPRO to get a little competition going.

```
$ VK_ICD_FILENAMES=/home/zmike/amd_pro.json ./vkoverhead -test 0 -output-only -duration 3
32889
```

what

[![mesa.png]({{site.url}}/assets/mesa.png)]({{site.url}}/assets/mesa.png)

What.

[![rotated_mesa.png]({{site.url}}/assets/rotated_mesa.png)]({{site.url}}/assets/rotated_mesa.png)

**THEY DARE?**

[![angry_mesa.png]({{site.url}}/assets/angry_mesa.png)]({{site.url}}/assets/angry_mesa.png)

## It's Totally Cool
...that AMDPRO is 15% faster than RADV. Yup, it's totally fine. No anger problems here, no sir, not with me, not even a little furious.

Cool as a cucumber.

But if—and this is obviously just a hypothetical—If I were enraged and just recovering from a lengthy tantrum after seeing these results, I'd be looking at making some artisanal spaghetti. To do that, I'd be running `perf` on the `vkoverhead` case and then checking out a flamegraph, which might even happen to look something like this

[![base.png]({{site.url}}/assets/spaghetti/base.png)]({{site.url}}/assets/spaghetti/base.png)

and you know it's weird that the graph would look like that since in a graph like that the actual emission of draw packets is only 18% of the CPU time, which means it's just *throwing away CPU cycles*, and no wonder the performance is worse, and I hate Wednesdays.

But again, don't ask if I'm okay, I'm completely fine, this isn't bothering me.

But if—and this is obviously just another hypothetical—If I'd just come back from a counseling session that was supposed to help me cope with these inferior performance results and wasn't feeling any better at all, then I'd definitely be craving some spaghetti. And so I'd be looking at [radv_emit_all_graphics_states()](https://gitlab.freedesktop.org/mesa/mesa/-/blob/eef1511437ac6173dfd202b2fc581860d161c183/src/amd/vulkan/radv_cmd_buffer.c#L7202) and [radv_upload_graphics_shader_descriptors()](https://gitlab.freedesktop.org/mesa/mesa/-/blob/eef1511437ac6173dfd202b2fc581860d161c183/src/amd/vulkan/radv_cmd_buffer.c#L3967) to see what the actual farfalle was going on with these fat pieces of stortini.

And the first of those functions, I'd see there were all kinds of null checks and branch chain disasters that were annihilating performance, so I'd probably [rip] and (https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=879428a94edf10c120c27e6365c6696038ce37eb)[tear](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=fd0d19c50ba3d5eb95b29c84473788a8f0be65fe) those right out, and then, just while I happened to be in the area, I'd [simplify some cache-killing indirect access](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=cc33f2bc51b82dad0589500d33c9702b70588af2), and, well, it's not like I'd leave without [clearing up those branches](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=ec9f1ab89f4f403b5bb65db7870fb50fdb319e99), right? Hah, of course not, though this is all just hypothetical anyway.

## I'm Not Being Defensive
Stop asking. I'm fine.

If I wasn't fine, I'd probably be running `vkoverhead` again at this point and seeing the following results

```
$ ./vkoverhead -test 0 -output-only -duration 3
36006
```

and then I'd be fine anyway since now RADV is up by 10%. Which is okay. It's not bad. Nothing to brag about, you know, just being up by such a tiny little amount over the competition, but it'll do.

[![1.png]({{site.url}}/assets/spaghetti/1.png)]({{site.url}}/assets/spaghetti/1.png)

Is what a responsible person would say.

But here at SGC, responsibility flies out the window when performance is involved, and I don't have enough spaghetti yet, so buckle in because this pasta machine is just getting started.

It's `perf` time again, and I've got another totally hypothetical flamegraph

[![states.png]({{site.url}}/assets/spaghetti/states.png)]({{site.url}}/assets/spaghetti/states.png)

which is less consumed by the stupidity of those fat pieces of stortini I insalted above, but I'm not in the mood for stortini at all today. They gotta go. `radv_upload_graphics_shader_descriptors()` I got my eye on you and your little [radv_flush_constants()](https://gitlab.freedesktop.org/mesa/mesa/-/blob/eef1511437ac6173dfd202b2fc581860d161c183/src/amd/vulkan/radv_cmd_buffer.c#L3510) too. Why is `radv_flush_constants()` even showing up here? What's the deal with that? There's no constants to flush. I'm taking 'em out.

Get the rolling pin, [flatten out the dough](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=57aa00baac1945c6b19033b780398e7c57f42d84), and what happens?

```
$ ./vkoverhead -test 0 -output-only -duration 3
38629
```

[![2.png]({{site.url}}/assets/spaghetti/2.png)]({{site.url}}/assets/spaghetti/2.png)

## Now We're Cooking
With `perf`, and I'm getting out another flamegraph, and it's better

[![flush.png]({{site.url}}/assets/spaghetti/flush.png)]({{site.url}}/assets/spaghetti/flush.png)

because of course it is. That draw packet emission is getting more time, the fat stortini is slimming down, and everything is great.

But does anyone out there actually think I'm about to stop now? When I'm only up by a tenuous 36% from where I started, and my lead over AMDPRO is a barely-noticeable 17%?

Take off your jacket, because I'm turning the heat of the burners up to high.

Look at this eyesore

[![bad.png]({{site.url}}/assets/spaghetti/bad.png)]({{site.url}}/assets/spaghetti/bad.png)

I'm about to end [this function's](https://gitlab.freedesktop.org/mesa/mesa/-/blob/eef1511437ac6173dfd202b2fc581860d161c183/src/amd/vulkan/radv_cmd_buffer.c#L3421) whole career. By [inlining it](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=6cfe62ac6b2dfa0d4a890ecbaec7070b881456e4).

```
$ ./vkoverhead -test 0 -output-only -duration 3
41878
```

[![3.png]({{site.url}}/assets/spaghetti/3.png)]({{site.url}}/assets/spaghetti/3.png)

## Finishing Touch
When serving any sort of dish, it's important to add a garnish. And you know what *isn't* a fucking garnish?

[![streamout.png]({{site.url}}/assets/spaghetti/streamout.png)]({{site.url}}/assets/spaghetti/streamout.png)

[This thing](https://gitlab.freedesktop.org/mesa/mesa/-/blob/eef1511437ac6173dfd202b2fc581860d161c183/src/amd/vulkan/radv_cmd_buffer.c#L9371).

So now [it's gone](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499/diffs?commit_id=955f48dd24e6a6735c967c1d1df52d0ae2926692) and what is the performance at now?

```
$ ./vkoverhead -test 0 -output-only -duration 3
44073
```

[![4.png]({{site.url}}/assets/spaghetti/4.png)]({{site.url}}/assets/spaghetti/4.png)

Incredible. The flavor (of winning), the atmosphere (of being a winner), the experience (of being #1), are all unparalleled.

This makes for a **55% increase in RADV's draw throughput** as well as a much more reasonable **30% lead over AMDPRO**.

All from growing just the right amount of spaghetti.