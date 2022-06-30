---
published: false
---
## Addressing Concerns

After my last post, I received a very real, nonzero number of DMs bearing *implications*.

That's right.

Messages like, "Nice blog post. I'm sure you totally know how to use RenderDoc." and "Nice RenderDoc tutorial." as well as "You don't know how to use RenderDoc, do you?"

I even got a message from Baldur "Dr. Render" Karlsson asking to see my RenderDoc operator's license. Which I definitely have. And it's not expired.

I just, uh, don't have it on me right now, officer.

But we can work something out, right?

Right?

## Community Service

So I'm writing this post of my own free will, and in it we're going to take a look at [a real bug](https://gitlab.freedesktop.org/mesa/mesa/-/issues/6240). Using RenderDoc.

Like a tutorial, almost, but less useful.

If you want a real tutorial, you should be contacting Danylo Piliaiev, Professor Emeritus at Render University, for his self-help guide. Or watch his [free zen meditation tutorial](https://www.youtube.com/watch?v=T2JQq1cxB1o&hd=1). Powerful stuff.

First, let's open up the renderdoc capture for this bug:

[![app.png]({{site.url}}/assets/renderdoc/app.png)]({{site.url}}/assets/renderdoc/app.png)

What's that? I skipped the part where I was supposed to describe how to get a capture?

Fine, fine, let's go back.

## RenderDoc + Zink: Not A HOWTO
