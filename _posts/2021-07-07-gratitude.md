---
published: false
---
## The Unsung Heroes

This is going to be less of a technical post and more of a *have you thought about* post. With that said, I think this is more important than the average post here, meaning that expectations should be set somewhere between *I need to stop everything else I'm doing until I finish reading* and *this is the most important event in my life*.

Let's talk about open source. No, Open Source. The idea of it.

## How Does Open Source Work?
Those of you who are veterans are rolling your eyes. Another post about the glory of Open Source.

The thing about Open Source is that it's sort of whatever you make of it. At its core, it's about getting people together to solve a problem—community building. Whether that community is large or small, the goal is the same: write some quality software.

To that end, you've got your usual corporate powerpoint slide of community roles:
* maintainers
* developers
* reviewers
* whatever other buzzwords are currently relevant

In Mesa, the *maintainer* and *developer* roles are mostly synonymous among core contributors: these are the people who write the code that gets posted about on all the news sites.

The *reviewer* is a bit more mysterious though. Who are reviewers, and what separates them from the others?

## WD-40
Reviewers are the grease that makes the project work. There's really no other way of saying it.

Outside of a few components of Mesa that are effectively the wild west, without any form of oversight or approval needed for changes to be landed, every driver and utility in the tree requires that changes undergo review before they land. This means that each and every patch which affects code or build has to have a person stop everything else they're doing and physically scroll through each patch, line-by-line, then add a *Reviewed-by* or *Acked-by* tag.

If you're unclear as to the meanings of these tags, consider it like you're going skydiving with someone you've never met before who has been in charge of preparing your parachute:
* **Reviewed-by** means "I triple-checked your parachute as well as your reserve, and I'm as certain as a human is capable of being that everything is how it should be"
* **Acked-by** means "Hey, I grabbed this already-packed parachute off the hanger and gave it a once-over; you'll probably be fine"

It's then up to the developer to decide whether to merge the code based on the feedback given to them by the reviewer.

This, of course, assumes they get feedback at all.

## Balance
Too often on news sites (and in certain corporate metrics) you'll see something like "Patches McCodesAlot, working for GreatCodingCompany, authored the most code changes for this release cycle (9001 patches), which is over 100x more than the next highest contributor.

The manager at the company sees this and thinks "I'll send this up the chain. We should poach Patches so we can have greater control over this project which underpins our entire business strategy. Also it'll make my powerpoint pie charts look rad."

The casual reader sees this and says "Wow, Patches is awesome! Without Patches, I probably couldn't even play Fororantwatch on my Linux gaming desktop!"

But how do the patches that Patches writes get merged into the release? Unless Patches works exclusively in one of the undermaintained areas of the project, in which case it's unlikely that their work is being widely used, the odds are that someone's pulling a huge lift on the review side to enable all of those patches into the repository.

This is the job of the reviewer.

## A Thanks
As this Mesa release cycle starts to wind down, I hope that readers of this blog and news sites can take a moment to look past Patches McCodesAlot and see the people who make it possible for Patches to land so many damn patches.

At the time of this post, this is what the top 10 reviewers managed to accomplish over the past few months:

