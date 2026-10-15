---
layout: post
title: LLM Thoughts #1
published: true
---

# The First, Probably Not The Last
Wall of text in coming. If we get through this, we can have funny memes again next time, promise.

Everyone's having thoughts on LLM use in coding these days, and being that SGC is at the forefront of all cutting-edge software development, obviously I have thoughts of my own. Thoughts that need to be shared. Get out of my head, thoughts.

I'll start off by stating my position: I'm pro-LLM use.

There's plenty of arguments against it at a "moral" level, but the same can be said for most technologies these days. Television, for example, is primarily used to indoctrinate the masses into taking detrimental stances on political/social issues. Brawndo, The Thirst Annihilator was a less harmful product than most advertisements shill for. I'm not dismissing those arguments. As a holder of a degree in Philosophy with a specialization in bioethics, this is literally what I trained for. But I'm not going to rage against the machine here because it's just not productive or useful.

In the future, we will all use LLMs. It's inevitable. Local models are already becoming competitive at the low-end, and within a year or two they'll likely be good enough to compete with what I'm using now.

Sidebar: I applied for Anthropic's "Claude for Open Source" usage. They did not accept my application. I therefore have nothing nice to say about Claude outside of it was kinda cool to have a free 7-day trial. Shame they missed out on that sponsorship opportunity cuz I've got some bangers come up and whichever LLM I'm actually using is doing awesome/unhinged work.

# The Basics: Positivity

Anyway, we'll all be using LLMs eventually. They're the new calculator. Why do long division by hand when you can just press some buttons and get the answer immediately? Sure, we learn how to do long division in school, and maybe if I stare at a piece of paper hard enough I can will myself into remembering how, but I'm never going to go back to those days of pencil and paper. I have tools, and they improve my life.

So far, my favorite use for LLMs is the branch rebase. Pull down the latest from git, slam that rebase button in the shell, and boom, a million conflicts. I've spent my hours manually resolving these, and it's not super fun. Low tier annoyance, sometimes high tier whenever I have to rebase across some giant new refactor/feature that touches all the things.

But now I can just tell my LLM "do the rebase and give me a recap". I then switch to another task (or get a snack) while it does the thing for me. This is a purely mechanical process, and I no longer have to do it.

Code review is also great. LLMs are an incredible static analyzer. I can tell it "consider XYZ. review this area of the tree. find issues related to XYZ." and it'll think for a while (maybe even an hour or two) and then come back to me with a report. I can even tell it to write new (piglit) tests. Coverage++.

I can point it at commits I'm working on and tell it to review them. Then humans have less of my shit code to wade through, and my future self has fewer regressions to bisect.

I can even point it at open MRs and tell it to review them for me. If the MR is entirely generated, I might save myself the time and just post the LLM's thoughts. If it's a human's work, I'm much more likely to read the machine's thinking, do my own analysis, and respond thoughtfully.

Best of all, I can throw it a link to a failing CI pipeline and tell it "figure this shit out", and it'll read the log, find the failing tests, build the branch, run the tests, debug them, and tell me how I fucked up.

These are all nice workflow and ecosystem improvements.

# Trickier: LLM Contributions
This is where everyone gets stuck. What happens when people start submitting "slop" MRs? What do we do about that? How do we keep from getting overwhelmed?

I'm gonna break this into a few different categories:
* drive-by patch submissions by enthusiasts to fix bugs
* larger patch submissions by enthusiasts for features/performance
* existing developers using LLM

First, I want to avoid the term "slop". I know we use it when we see AI slop memes, and that's fine, and AI slop art is what it is, but just as I don't think code is art (come at me bro),

[![code-not-art.png]({{site.url}}/assets/code-not-art.png)]({{site.url}}/assets/code-not-art.png)

I also don't think that LLM-generated code is slop. It's all just code, and it compiles into machine stuff so the thinking sand can render triangles for me.

I also want to avoid witch-hunting the "non-developer class" of contributors using LLMs to submit patches. I like the enthusiasm. Genuinely. It brings me back to ye olden days of open source before I became a jaded husk. That's why I call them enthusiasts. That's what they are. I may or may not accept the patches they submit, but I don't think less of them for trying to contribute. I was once one of them, yearning to be able to comprehend these files full of arcane arrows and punctuation.

## Drive-by Bugfix Patches From Enthusiasts
I've had a flurry of these over the past week. I can only imagine that my big-endian removal shitpost inspired some people to take their problems into their own hands.

I love this, and I think it's great.

Before, whenever we had a bug, we just had to sit and rage at our PCs, maybe open a ticket if we were upset enough, or if we were thoroughly furious we'd start our own competing project just to say fuck-you to whatever thing we were using that didn't work how we wanted. Now, we can tell our computer "figure this shit out for me" and get something that fixes the problem. Maybe it doesn't fix it the "right" way, as described in Old Testament, but it's good enough for government.

The next step is to take that fix and throw it upstream. Maybe it's good enough. Maybe it's not. We don't know, and honestly maybe we don't care that much; we're just trying to help.

That's how I view the perspective here. So when I see this sort of patch come in, I review it accordingly. Most of the time they get my stamp of approval because zink has no bugs, and when it does have bugs they're usually tiny little micro-bugs that only need a one-liner.

Saves me time, fixes real issues, awesome. Keep 'em coming. But know that you gotta pay the piglit tax any time you find a new bug.

## Larger Patch Submissions By Enthusiasts
This is where it starts to get tricky. I'm not referencing any particular MR or patchset here, but if I see a big LLM-generated/assisted patchset from someone I don't know, I get antsy. And if you aren't putting the LLM trailers on your patches, then don't worry: I can still tell you're vibecoding because you're submitting a zink patch and I've spent most of a decade writing the kind of spaghetti code that no organic human can comprehend.

The reason this is tricky is that now I'm "reviewing" code that a human didn't write. So why am I bothering to review it? Just like I don't think code is "art", I don't think LLM code is "work".

[![code-not-work.png]({{site.url}}/assets/code-not-work.png)]({{site.url}}/assets/code-not-work.png)

I've done a lot of vibecoding the past couple weeks, so I know the actual human effort required to churn out lines of code. If an enthusiast submits a patchset they don't understand, I'm effectively just talking to a machine that has a high-latency meat proxy.

At that point, I may as well just close the MR and tell my own machine to write code for the same thing. Then I can directly interact with the output. LLM-generated code isn't "owned" by anyone (unless courts decide to start taking real action about training materials lmaohahahahahaha), so even if I take your submission, tell my agent to rework it, and slap my authorship on it, that's my prerogative.

I don't love this. It doesn't fill me with joy or enthusiasm.

If you find yourself thinking about doing this, **honestly don't**. Don't do it. Please. Just create an issue on the tracker requesting that I do it. I'm the human that has to review the end result anyway, and it's going to be way faster and easier for all of us if I don't burn out in the process by using some shitty gitlab-RPC to tell an LLM what I really want. If we get down to my LLM reviewing your LLMs 500-patch series over gitlab, we have failed.

Sidebar: I fucking hate gitlab now. How/why is it so bad? Should I vibecode a replacement or just send them patches to fix it?

## Existing Developers Using LLM
The final frontier. What happens when we're all doing it?

I think this comes down to the human element. At the end of the day, there is a real human behind all of our hardened developer facades. If I know you and you're throwing LLM code up into the repo, that's fine. I can have some degree of confidence that your 500-patch series has been driven/steered by you, so I can have the confidence that you will understand my review comments and be able to consider how best to steer your LLM towards a solution we can all agree on. And you won't be insulted when I tell you what the actual fuck is this shit did you even read any of this?

My only ask is that you don't burn me out by chucking a shitload of huge MRs over the wall at once. Pace yourself. Keep your MRs to a reasonable, reviewable size in areas you possess organic competency. Humans are still the bottleneck here.

# Closing Thoughts
I'm not going to make it official or anything, but this is roughly where I'm at as far as a policy for zink or any other stuff I maintain. I'm not a big fan of official policy. I don't trust governance or governments. They're out to get me. Fluoride in drinking water is a psyop to make us ignore how James Buchanan has been secretly running the global political system from his base on the moon since 1862. Look it up.

That said, I think it's valuable to keep the `Assisted-by` and `Generated-by` trailers, though I'm not keen on becoming a billboard for exact models. We need accountability. If, a year from now, I find out that the model I'm currently using produced a bunch of code that has bugs, having a tag on commits will let me do deeper review on suspect patches. If I see a wild drive-by patch on my dashboard, knowing up-front whether I'm interacting with a real person or a LLM's meat interface helps me determine how to handle the review. Alternatively, someone trying to conceal LLM use from me gives me a different inclination of how to handle the review.

If I see a car veering unexpectedly into my lane, I want to know that it's a dog at the steering wheel.

The steering is important. Anyone and everyone can use LLMs to vibecode whatever they want now, but that doesn't mean we should immediately yolo whatever comes out the pipe into the repo. As an example, one project I've been working on recently involved CPU optimizations related to draw calls. The initial version is in the top few commits of [this branch](https://gitlab.freedesktop.org/zmike/mesa/-/commits/zink-drawfast). You can clearly see where it was going with this: eliminate as many instructions as possible by creating an entirely bespoke draw function to handle the special case. Massive instruction savings—the result looks great.

Then check out [the version I actually went with](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/45263). This is a radically different version which provides equivalent results, and it does so without creating any bespoke special-case functions. Moreover, it improves the general-case performance of draws, not just a single case.

I think the difference between these two cases exemplifies the difference between "good" AI use and "bad" AI use. It took me a long time to get from A to B, even though I wasn't personally writing the code (still faster than if I was tho), but with enough domain knowledge I could steer my way towards the better solution even after the LLM claimed it was done.

The goal of using any tool should be to get better results while improving QoL for the human using it.
