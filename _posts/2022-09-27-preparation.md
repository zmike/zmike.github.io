---
published: false
---
## Shoutouts

Big thanks to Timur for taking over merging all my misc RADV patches. I'm so far underwater I can see the bottom of the Mariana Trench, and without his help, these likely wouldn't be landing until 2023.

## Cooking Up A Storm

It's been a while, but today's blog is a zink blog. What's been happening?

Well, if you've been following prominent news sites, you might not think there's too much happening on the performance front. And whew do I have news for you.

Big news.

With XDC next week, I'm throwing down the gauntlet.

Mesa 22.3 is going to be the **BIG PERF RELEASE**.

I said it.

## No Memes
Could not be more serious.

Don't believe me? Bam, have another 20-30% performance on Turnip I "quietly" [merged](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18358) a month ago. And also some amount of improvement on Intel that I haven't quantified yet. Who could've guessed that enabling compression on images would have benefits?

Seeing mega CPU usage? [Not](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18135) if I've [got](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18637) [anything](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18364) to [say](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18499) about [that](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18786). Still more to come here too.

Your game/emulator randomly freezes for a few seconds on startup? I've [heroically](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18198) made the PBO download ubershaders compile without blocking. Also there's new, optimized variants that perform even better.

What's that? Zink is still unusable for gaming because of all the shader compilation stuttering? [Blammo](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18197), your compute shaders are now precompiling in the background.

Oh, but you're *still* seeing stuttering because of shader compiles? Not for long, because I'veOH SHIT KHRONOS IS HERE I'M SORRY I DIDN'T THINK IT'D BE A BIG DEAL TO