# I <3 Open Source
That should be obvious by now, right? I've been out here blogging about Open Source stuff for over a decade, and occasionally I still have time to actually write code.

Haha. But seriously.

I believe in Open Source. I believe in the collaborative development model, the teamwork, the shared vision of contributing to projects that can stand up to and even surpass big-name proprietary products.

I believe in getting shit done. I believe in discussion, in deliberation, in review, but at the end of the day I also believe that people need to be realistic and accept compromises rather than dying on every fucking hill of a gitlab review comment. I believe in it enough that I'm sleeping five or fewer hours a night this week.

And believe it or not, [I'm also a Wayland developer](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/stable/xdg-shell/README).

# Conflict: Just Part of the Process
I work for Valve. I've [blogged about]({{site.url}}/unsung/) what it's like working for Valve, and nothing has changed since then. It's still great, and Rhys Perry (not you, the other one) is still an unsung hero.

As I mentioned in that post, one of the great things about my job is the freedom to improve projects without managerial overhead. To self-determine. To see the goal in the distance and get there in a way that suits me.

I'm not the only person working for Valve, and I'm not the only one who enjoys this freedom. By now, everyone has seen the latest happenings with regards to the [efforts to improve Wayland display mechanisms](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/31329) around refresh rates and swapchain-related stuttering. I sympathize with the frustration radiating out of this effort. I see myself in it; this is someone who saw a problem, saw a goal in the distance, and chose a path forward that suited them.

I don't believe there was malice or ill-will involved here, just growing frustration at a long-standing problem that was defying efforts to resolve it. I've been there. We've all been there. Everyone has had at least some Open Source interaction where a review has stalled, or gotten heated, or failed to progress in finite time for some reason.

It happens to be the case that wayland-protocols, as a project, has these sorts of interactions more often than other projects. Again, I don't believe there is malice or ill-will involved. Maybe I'm optimistic, or an idealist, or whatever, but I believe everyone participating in Wayland development wants it to succeed. Maybe they tunnel-vision too much, maybe they get sidetracked and disappear, maybe they get heated because *WHY CAN'T YOU JUST UNDERSTAND THAT I KNOW WHTA I'M TALKNG ABOUT??!!1*, maybe they post long walls of text that nobody really wants to read but inevitably you know there's some kind of a point hidden somewhere in there so you gotta just reach in and tweeze it out with your fingers like it's stuck between the cushions of your sofa so you can tell them what an idiot they are, etc. Again, we've all been there.

This is how Open Source works, though. This is how the sausage is made. It's not always pretty. It's not always easy. It's not free as in freedom or beer, it's [free as in puppies](https://opensource.com/article/17/2/hidden-costs-free-software): sometimes they just do what they want, and you gotta be okay with that.

# Resolutions
Open Source has disagreements, and I'm okay with that. I still believe that at the end of the day, everyone is capable of stepping back, seeing the goal, and arriving at some sort of agreement that gets us all there together.

It's for that reason I've [proposed some updates](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/339) to wayland-protocols governance to address some of the systemic issues that have persisted since before there was official 'governance'. I agree that the system is a little broken, but I don't think the solution is to throw out the whole system.

**We can't just throw away puppies.** Well, we can, but it's complicated and people might get the wrong idea. We gotta have a real good reason to throw those puppies out on the street. If we don't at least say we tried to make it work, to get them to stop chewing on our shoes, and peeing on the sofa, and eating the sandwiches we forget about because we gotta go back and argue with those idiots in Mesa who can't understand the brilliance of our latest idea to ship OpenGL ray-tracing--if we can't at least show that we cannot fundamentally coexist with these puppies, then *some people* might think we aren't capable of being good dog owners. They might wonder whether their puppies are safe around us, or worse: whether their GPUs are.

The problems facing wayland-protocols are many, and my blogging time is (somehow) not infinite. In short:
* new stuff hard
* `stable/` stuff harder

The first problem is more tractable, and it's the one causing the most frustration at present, so that's where I started. An inability to land new protocols, even just for broader development purposes, hurts the ecosystem, both by stifling innovation and frustrating would-be contributors. Many solutions have been proposed over the years, though few have been official. There [was one from @emersion](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/192) last year that sort of almost gained traction but then also wasn't quite what people wanted. There was also...

That's it, actually. That's the only time an official proposal has been made to change the governance in a substantial way. Which is to say that though everyone knew and acknowledged problems exist, nobody (else) took the action to try solving it, as plainly spelled out by [section 4.1](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/GOVERNANCE.md) of the governance model document.

It's Open Source all the way down: Just create a merge request**\***.
**\*** Yes, I know it says only members can propose changes, but surely someone could have wrangled one of the members and drafted something? Surely the governance members would react positively to a good-faith proposal made by a non-member? We don't *have* to act like every other person on the planet is out to get us, do we?

To be clear, I'm not blaming anyone.

I get it.

Complaining about hard problems is way easier than fixing them. I complain a lot too. That's why I have this blog, where I can complain about spaghetti, bad memes, and drivers ~~that are faster than mine~~that have yet to be surpassed—you know what SGC is about by now.

At the end of the day, however, sometimes this is what peak Open Source performance looks like:

[![dog-walking.png]({{site.url}}/assets/dog-walking.png)]({{site.url}}/assets/dog-walking.png)