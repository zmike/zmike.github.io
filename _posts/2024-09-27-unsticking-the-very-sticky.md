# Day 4 of Wayland governance hacking
I wake at 5 AM. This is the perfect time to wake up in NYC TZ, as it affords me the ability to eat a whole
apple in the time it takes my little internet-browsing chromebook to load all the IRC and Discord backlogs from
the five hours that I snuck away for a nap when nobody was watching.

I slather the apple with a haphazard scoop of peanut butter; getting away from a keyboard for more than twenty six
seconds in a given stretch is difficult, and I need protein. While entering into a fraught negotiation over
the meaning of `30-day discussion period` with my left hand, I carefully scoop protein powder into a shaker with my right.
There's no time to waste. Not even a single second--Another argument could break out, steal a 1973 Pontiac Firebird, and
go joyriding on the wrong side of the freeway.

I'm writing this blog post with my toes. They know their way around a keyboard, but they're slow and prone to mistakes.
My cat is in charge of hitting an oversized backspace key when I dangle his favorite toy over it. It'll be hours
before we get something together that can be read coherently.

This is my life now.

This is what it takes to do Open Source.

# Final Day: Everything, Everywhere, All At Once
I've put up a couple sizable proposals to resolve longstanding issues and oversights in the governance model. Today is Friday,
however, which means it's the final day. Once we hit the weekend, everyone will collectively fuck off and forget everything
that happened this week, which means I have to maintain peak velocity and finish strong.

Let's fucking go.

[Last proposal](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/342).

## Problem 1: HOW IS THIS %#$@$#@#$%%$ PROTOCOL STILL STUCK AFTER 4 YEARS?!?!?!?!?
It's a great question. I asked it myself. The answers are myriad and nebulous, but I'm the guy who explains things, so I'm gonna
break it down.

Imagine you're wayland-protocols. You've got all these puppies. And you're walking them--so you tell yourself, but really they're walking themselves.
They're walking you. And they're going in whatever direction they want. And out of all these puppies you've got two,
one's trying to go left to chase a car, and the other one's trying to sniff a telephone pole on the right. The other
fifty seven puppies just want to keep moving because they love their walkies. But these two puppies are the biggest ones,
and they're pulling the others along with them. So now your leashes are getting all tangled, and you're being dragged around,
and everyone's pointing at you because you look like you don't know what you're doing.

That's where we're at now. Everyone's laughing at you.

Look at this *idiot* trying to walk fifty nine puppies at once. This absolute *moron*. Who would ever do that? Why not just walk
one or maybe two puppies at a time like everyone else? That's the way you're *supposed* to walk them. The way people have *always* walked them.

But you know what? Walking fifty nine puppies individually would take all day. Nobody has the time to walk fifty nine puppies individually no
matter how cute or eloquent they are. So you need some way to resolve this. Or something.

Look, you get where I'm going with this.

Wayland protocol discussions get bogged down by people throwing out hypotheticals that can't truly be resolved, or by people talking past
each other, or by people disappearing, or the phase of the moon, or any number of reasons, and there's no official way to get
past these blockages. That's why I'm proposing tie-breaker votes as a simple way of moving past these problems when they arise.

Everyone understands tie-breakers: you vote, and the side with the most votes wins. It's that simple.

In this context, the wayland-protocols [member projects](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/MEMBERS.md)
vote (with one of them representing the author for non-members) and the majority wins. If there's another tie, the author gets to break it.

Simple. Done.

## Problem 2: Perfect Is the Enemy of Good
Sometimes a protocol in `staging/` is "good enough". The author has checked out, people are using the protocol,
and everyone is happy with it.

But it's still not a `stable/` protocol.

In this scenario, after an extended period of time without changes, any `staging/` protocol can be nominated by a
[member project](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/MEMBERS.md) for `stable/` promotion.
Some discussion happens, and then it becomes stable.

Simple. Done.

## Problem 3: Start Times
The governance model talks about discussion periods, but it doesn't specify exactly when they begin. For example, on any of my governance MRs,
does the 30-day period start when I open the MR or when the MR is approved?

Obviously it starts when I open the MR. We gotta keep things moving.

Done.

## Problem 4: Project Representation
The governance document specifies that a [member project](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/MEMBERS.md)
may have up to two official representatives. This can be problematic, as it puts pressure on 1-2 people to be on top of
every active protocol discussion.

Instead, projects should be represented by as many individuals as they want (pending the usual process for adding points-of-contact). This
ensures that protocols don't get blocked waiting for a given project to take a look when all representatives are busy. It also helps more diverse
projects (e.g., wlroots) ensure that opinions from more of its constituents are officially represented.

Each project still only gets one vote, but now that vote can be more readily deployed and voiced.

# I think we're done here?
From what I've seen, this should cover all the major issues that have been negatively impacting Wayland development. Sure, there are other,
more minor issues, but I'm not aware of anything that can't be solved through good old person-to-person discussion.

Maybe all this works, and maybe it doesn't. But at least now if we decide to throw away some puppies, nobody can question whether we
really tried *everything*.