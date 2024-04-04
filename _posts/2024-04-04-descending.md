# Into The Spiral of Madness

I know what you're all thinking: there have not been enough blog posts this year. As always, my highly intelligent readers are right, and as always, you're just gonna have to live with that because I'm not changing the way anything works. SGC happens when it happens.

And today. As it snows in April. SGC. Is. Happening.

Let's begin.

# In The Beginning, A Favor Was Asked
I was sitting at my battlestation doing some very ordinary **REDACTED** work for **REDACTED**, and friend of the blog, Samuel "Shader Objects" Pitoiset (he has legally changed his name, please be respectful), came to me with a simple request. He wanted to enable [VK_EXT_shader_object](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_shader_object.html) for the radv-zink jobs in mesa CI as the final part of his year-long bringup for the extension. This meant that all the tests passing without shader objects needed to also pass with shader objects.

This should've been easy; it was over a year ago that the Khronos blog famously and confusingly [announced](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today) that pipelines were dead and nobody should ever use them again (paraphrased). A year is more than enough time for everyone to collectively get their shit together. Or so you might think.

Turns out shader objects are hard. This simple ask sent me down a rabbithole the likes of which I had never imagined.

It started normally enough. There were a few zink tests which failed when shader objects were enabled. Nobody was surprised; I wrote the zink usage before validation support had landed and also before anything but lavapipe supported it. As everyone is well aware, lavapipe is the best and most handsome Vulkan driver, and just by using it you eliminate all bugs that your application may have. RADV is not, and so there are bugs.

A number of them were simple:
* [invalid location assignment for patch variables](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10414)
* harmless [VVL spam](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10457) about missing dynamic state setters
* Vulkan spec had [broken corner cases](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10737)
* [reporting wrong value](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10750) for max patch components
* also patch variables were differently broken in [another way](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28296)

The list goes on, and longtime followers of the blog are nodding to themselves as they skim the issues, confirming that they would have applied all the same one-liner fixes.

Then it started to get crazy.

# Locations, How Do They Work?
I'm a genius, so obviously I know how this all works. That's why I'm writing this blog. Right?

[![smart-or-blogger.png]({{site.url}}/assets/smart-or-blogger.png)]({{site.url}}/assets/smart-or-blogger.png)

Right. Good. So Samuel comes to me, and he hits me with [this absolute brainbuster](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10809) of an issue. An issue so tough that I have to perform an internet search to find a credible authority on the topic. I found this amazing and informative [site]({{site.url}}/adventures-in-linking) that exactly described the issue Samuel had posted. I followed the staggering intellect of the formidable author and blah blah blah yeah obviously the only person I'd find writing about an issue I have to solve is past-me who was too fucking lazy to actually solve it.

I started looking into this more deeply after taking a moment to [fix](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28185) a different issue related to location assignment that Samuel was too lazy to file a ticket for and thus has deprived the blog of potential tests that readers could run to examine and debug the issue for themselves. But the real work was happening elsewhere.

# Deeper
Now we're getting to the good stuff. I hope everyone has their regulation-thickness [safety helmet](https://en.wikipedia.org/wiki/Tin_foil_hat) strapped on and splatter guards raised to full height because you'll need them both.

As I said in **Adventures In Linking**, `nir_assign_io_var_locations` is the root of all evil. In the case where shaders have mismatched builtins, the assigned locations are broken. I decided to take the hammer to this. I mean I took the forbidden action, did the very thing that I railed about live at XDC.

Sidebar: at this exact moment, Samuel told me his issue was already fixed.

I added a new pipe cap.

I know. It was a last resort, but I wanted the issue fixed. The result was [this MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28265), which gave `nir_assign_io_var_locations` the ability to ignore builtins with regard to assigning locations. This would resolve the issue once and for all, as drivers which treat builtins differently could pass the appropriate param to the NIR pass and then get good results.

Problem ~~solved~~.

# Deeper.
I got some review comments which were interesting, but ultimately the problem remained: lavapipe (and maybe some other vulkan drivers) use this pass to assign locations, and no amount of pipe caps will change that.

It was a tough problem to solve, but someone had to do it. That's why I dug in and began examining [this MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/26819) from the only man who is both a Mesa expert and a Speed Force user, Marek Olšák, to enable his [new NIR optimized linker](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8841) for RadeonSI. This was a big, meaty triangles-go-brrr thing to sink my teeth into. I had to get into a different headspace to figure out what I was even doing anymore.

[![code-motion.png]({{site.url}}/assets/code-motion.png)]({{site.url}}/assets/code-motion.png)

The gist of `opt_varyings` is that you give all the shaders in a pipeline to Marek, and Marek says "trust me, buddy, this is gonna be way faster" and gives you back new shaders that do the same thing except only the vertex shader actually has any code. Read the design document if you want more info.

Now I'm deep into it though, and I'm reading the commits, and I see there's this new `lower_mediump_io` callback which lowers mediump I/O to 16bit. Which is allowed by GLSL. And I use GLSL, so naturally I could do this too. And I did, and I ran it in zink, and I put it through CTS and OH FUCK OH SHIT OH FUCK WHAT THE FUCK EVEN--

[![mediump.png]({{site.url}}/assets/mediump.png)]({{site.url}}/assets/mediump.png)

# mediump? More Like... Like... Medium... Stupid.

Here's the thing. In GLSL, you can have mediump I/O which drivers can translate to mean 16bit I/O, and this works great. In Vulkan, we have this knockoff brand, dumpster tier [VK_KHR_16bit_storage](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_16bit_storage.html) extension which *seems* like it should be the same, except for one teeny tiny little detail:

```
• VUID-StandaloneSpirv-Component-04920
  The Component decoration value must not be greater than 3
```

Brilliant. So I can have up to four 16bit components at a given location. Two whole dwords. Very useful. Great. Just what I wanted. Thanks.

Also, XFB is a thing, and, well, pardon my saying so, but mediump xfb? Fuck right off.

# Next Up: IO Lowering—FRONTEND EDITION
With mediump safely ejected from the codebase and my life, I was free to pursue other things. I didn't, but I was free to. And even with Samuel screaming somewhere distant that his issue was already long since fixed, I couldn't stop. There were other people struggling to implement `opt_varyings` in their own drivers, and as we all know, half of driver performance is the speed with which they implement new features. That meant that, as expected, RadeonSI had a significant lead on me since I'm always just copying Marek's homework anyway, but the hell if I was about to let some other driver copy homework faster than me.

Fans of the blog will recall way, way, way, way back in Q3 '23 when I blogged about [very dumb things]({{site.url}}/Dumber/). Specifically about how I was going to start using "lowered I/O" in zink. Well, I did that. And then I let the smoking rubble cool for a few months. And now it's Q2 '24, and I'm older and unfathomably wiser, and I am about to put this rake into the wheel of my bicycle once more.

In this case, the rake is `nir_io_glsl_lower_derefs`, which moves all the I/O lowering into the frontend rather than doing it manually. The result is the same: zink gets lowered I/O, and the only difference is that it happens earlier. It's less code in zink, and...

[![frontend-doit.png]({{site.url}}/assets/frontend-doit.png)]({{site.url}}/assets/frontend-doit.png)

Of course there is no driver but RadeonSI which sets `nir_io_glsl_lower_derefs`.

[![clown.png]({{site.url}}/assets/clown/1.png)]({{site.url}}/assets/clown/1.png)

And, of course, RadeonSI doesn't use any of the common Gallium NIR passes.

[![clown.png]({{site.url}}/assets/clown/2.png)]({{site.url}}/assets/clown/2.png)

But surely they'd still work.

[![clown.png]({{site.url}}/assets/clown/3.png)]({{site.url}}/assets/clown/3.png)

Surely at least some of them would work.

[![clown.png]({{site.url}}/assets/clown/4.png)]({{site.url}}/assets/clown/4.png)

Surely there wouldn't be that many of them.

[![clown.png]({{site.url}}/assets/clown/5.png)]({{site.url}}/assets/clown/5.png)

Surely ~~fucking all of them~~the ones that didn't work would be easy to fix.

[![clown.png]({{site.url}}/assets/clown/6.png)]({{site.url}}/assets/clown/6.png)

Surely they wouldn't uncover any other, more complex, more time-consuming issues that would drag in the entire Mesa compiler ecosystem.

[![clown.png]({{site.url}}/assets/clown/7.png)]({{site.url}}/assets/clown/y.png)

Wouldn't be worth mentioning at SGC if any of those were true, would it.

# SGC vs Old NIR Passes
By now I was pretty deep into this project, which is to say that I had inexplicably vanished from several other tasks I was supposed to be accomplishing, and the only way out was through. But before I could delve into any of the legacy GL compatibility stuff, I had bigger problems.

Namely everything was exploding because I failed to follow the directions and was holding `opt_varyings` wrong. In the fine print, the documentation for the pass very explicitly says that `lower_to_scalar` must be set in the compiler options. But did I read the directions? Obviously I did. If you're asking whether I read them comprehensively, however, or whether I remembered what I had read once I was deep within the coding fugue of fixing this damn bug Samuel had given me way back wh

With `lower_to_scalar` active, I actually came upon the big problem: my existing handling for lowered I/O was inadequate, and I needed to make my code better. Much better.

Originally when I [switched to lowered I/O](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24634), I wrote some passes to unclown I/O back to variables and derefs. There was one NIR pass that ran early on to generate variables based on the loads and stores, and there was a second that ran just before spirv translation to convert all the load/store intrinsics back to load/store derefs. This worked great.

But it didn't work great now! Obviously it wouldn't, right? I mean, nothing in this entire compiler stack ever works, does it? It's all just a giant jenga tower that's one fat-finger away from total and utter—What? Oh, right, heh, yeah, no, I just got a little carried away remembering is all. No problem. Let's keep going. We have to now that we've already come this far. Don't we? I'll stop writing if you stop reading, how about that. No? Well, heh, of course it'd be that way! This is... We're SGC!

So I had this `rework_io_vars` function, and it. was. BIG. I'm talking [over a hundred lines](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24634/diffs?commit_id=9e42553ca8d30a2a2cb6781774631c45285d77dd#fda22e5538476c03927b35602949859b6129ca82_4826_5139) with loops and switches and all kinds of cool control flow to handle all the weird corner cases I found at 4:14am when I was working on it. The way that it worked was pretty simple:
* scan through the shader looking for loads/stores
* using the load/store instruction's component type/count, infer a variable
* pray that nothing with complex indirect access comes along

It worked great. Really, there were no known bugs.

The problem with this came with the scalarized frontend I/O lowering, which would create patterns like:
* `store(location=1, component_count=1)`
* `store(location=0, component_count=1, array_size=4, array_offset=$val)`

In this scenario, there's indirect access mixed with direct access for the same location, but it's at an offset from the base of the array, and it kiiinda almost works except it totally doesn't because the first instruction has no metadata hint about being part of the second instruction's array. And since the pass iterates over the shader in instruction order, encountering the instructions in this order is a problem whereas encountering them in a different order potentially wouldn't be a problem.

I had two options available to me at that point. The first option was to add in some workarounds to enlarge the scalar to an array<scalar> when encountering this pattern. And I tried that, and it worked. But then I came across a slightly different variant which didn't work. And that's when I chose the second option.

Burn it all down. The whole thing.

I mean, uh, just—just that one function. It's not like I want to **BURN THE WHOLE THING DOWN** after staring into the abyss for so long, definitely not.

The new pass! Right, the new pass. The new `rework_io_vars` pass that I wrote is a sequence of operations that ends up being far more robust than the original. It works something like this:
* First, rely only on the `shader_info` masks, e.g., `outputs_written` and `inputs_read`
* `rework_io_vars` is the base function with special-casing for VS inputs and FS outputs to create variables for those builtins separately
* With those done, check for the more common I/O builtins and create variables for those
* Now that all the builtins are done, scan for indirect access and create variables for that
* Finally, scan and create variables for ordinary, direct access

The "scan" process ends up being a function called `loop_io_var_mask` which iterates a `shader_info` mask for a given input/output mode and scans the shader for instructions which occur on each location for that mode. The gathered info includes a component mask as well as array size and fbfetch info--all that stuff. Everything needed to create variables. After the shader is scanned, variables are created for the given location. By processing the indirect mask first, it becomes possible to always detect the above case and handle it correctly.

Problem ~~solved~~.

# Problems Only Multiply
But that's fine, and I am so sane right now you wouldn't believe it if I told you. I wrote this great, readable, bulletproof variable generator, and it's tremendous, but then I tried using it without `nir_io_glsl_lower_derefs` because I value bisectability, and obviously there was zero chance that would ever work so why would I ever even bother. XFB is totally broken, and there's all kinds of other weird failures that I started examining and then had to go stand outside staring into the woods for a while, and it's just not happening. And `nir_io_glsl_lower_derefs` doesn't work without the new version either, which means it's gonna be impossible to bisect anything between the two changes.

Totally fine, I'm sure, just like me.

By now, I had a full [stack](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28466) of zink compiler cleanups and fixes that I'd accumulated in the course of all this. Multiple [stacks](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28530), really. So many [stacks](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28162). Fortunately I was able to slip them into the repo without anyone noticing. And also without CI slowing to a crawl due to the freedreno farm yet again being in an absolute state.

I was passing CTS again, which felt great. But then I ran piglit, and I remembered that I had skipped over all those Gallium compatibility passes. And I definitely had to go in and fix them.

[![mesa-doge.png]({{site.url}}/assets/mesa-doge.png)]({{site.url}}/assets/mesa-doge.png)

There were a lot of these passes to fix, and nearly all of them had the same two issues:
* they only worked with derefs
* they didn't work with scalarized I/O

This meant I had to add handling for lowered I/O without variables, and then I also had to add generic handling for scalarized versions of both codepaths. Great, great, great. So I did that. And [one of them](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28462) really needed a lot of work, but most of the others were [reasonably straightforward](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28463).

And then there's `lower_clip`.

`lower_clip` is a pass that rewrites shaders to handle user-specified clip planes when the underlying driver doesn't support them. The pass does this by leveraging clipdistance.

And here's the thing about clipdistance: unlike the other builtins, it's an array. But it's treated like a vector. Except you can still access it indirectly like an array. So is it an array or is it a vector? Decades from now, graphics engineers will still be arguing about this stupidity, but now is the time when I need to solve this, and it's not something that I as a mere, singular human, can possibly solve. Hah! There's no way I'd be able to do that. I'd have to be crazy. And I'm... Uh-oh, what's the right way to finish that statement? It's probably fine! Everything's fine!

But when you've got an array that's treated like a vector that's really an array, things get confusing fast, and in NIR there's the `compact` flag to indicate that you need to reassess your life choices. One of those choices needing reassessment is the use of `nir_shader_gather_info`, a simple function that populates `shader_info` with useful metadata after scanning the shader. And here's a pop quiz that I'm sure everyone can pass with ease after reading this far.

How many shader locations are consumed by `gl_ClipDistance`?

Simple question, right? It's a variably-sized float[] array-vector with up to 8 members, so it consumes up to two locations. Right? No, that's a question, not a rhetorical—But you're using `nir_shader_gather_info`, and it sees `gl_ClipDistance`, okay, so how many slots do you expect it to add to your `outputs_written` bitmask? Is it 8? Or is it 2? Does anybody really know?

Regardless of what you thought, the answer is 8, and you'll get 8, and you'll be happy with 8. And if you're trying to use `outputs_written` for anything, and you see any of the other builtins within 8 slots of `gl_ClipDistance` being used, then you should be able to just *figure it out* that this is clipdistance playing pranks again. Right?

[![clipdistance-ohyou.png]({{site.url}}/assets/clipdistance-ohyou.png)]({{site.url}}/assets/clipdistance-ohyou.png)

*It's all fun and games until someone gets too deep into clipdistance* is a proverb oft-repeated among compiler developers. Personally, I went back and forth until I cobbled together [something](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28554) to sort of almost fix the problem, but I [posed the issue](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10925) to the community at large, and now we are having [plans with headings and subheadings](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10948). You're welcome.

And that's the end of it, right?

# Nope

The problem with going in and fixing anything in core Mesa is that you end up breaking everything else. So while I was off fixing Gallium compatibility passes, specifically `lower_clip`, I ended up [breaking freedreno and v3d](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28308). Someday maybe we'll get to the bottom of that.

But I'm fast-forwarding, because while I was working on this...

What even is *this* anymore? Right, I was fixing Samuel's bug. The one about not using `opt_varyings`. So I had my variable generator functioning, and I had the compat passes working (for me), and CTS and piglit were both passing. Then I decided to try out `nir_io_glsl_opt_varyings`. Just a little. Just to see what happened.

I don't have any more jokes here. It didn't work good. A lot of things went boom-boom. There were some `opt_varyings` bugs [like](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28304) [these](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28431), and some related bugs like [this](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28436), and there was missing core NIR stuff for [zink](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28429), and there were [GLSL bugs](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28529), and also [CTS was broken](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10361). Also a bunch of the earlier zink stacks of compiler patches were fixing bugs here.

But eventually, over weeks, it started working.

# The Deepest Depths
Other than verifying everything still works, I haven't tested much. If you're feeling brave, try out [the MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28580) with dependencies (or wait for rebase) and tell me how the perf looks.

Finally, it's over.
  
Samuel, your bug is fixed. Never ask me for anything again.