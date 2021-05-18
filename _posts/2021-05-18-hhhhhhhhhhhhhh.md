---
published: false
---
## Click Play

It's been a while.

I meant to blog. I meant to make new zink-wip snapshots. I meant to shower.

Look, none of us are perfect, and I'm just gonna get into some graphics so nobody remembers how this post started.

[![tombraider-suballocated.png]({{site.url}}/assets/tombraider-suballocated.png)]({{site.url}}/assets/tombraider-suballocated.png)

Boom, beautiful triangles. Look at that ultra smooth fps in mangohud. Protip: if you're seeing weird flickering or misrenders in your app/game, try throwing mangohud in front of the zink bus to see if it fixes them.

So what has been going on for the past however since the last post?

In a word: lots.

Here's the rundown.

## The Rundown
The 20210517 zink-wip snapshot is the bigest one in history. I say this with no exaggeration.

Changes since the last snapshot include:
* an imperial units (and I measured this precisely) fuckton of general driver overhead reduction
* (yet another) queue/dispatch rewrite, this one more optimized for threaded and multi-context use
* an actually working disk cache implementation
* an entire suballocator

One way or another, this is going to feel like a new driver. Ideally I'll be doing a post every day detailing one of the items on that list, but for now I'll close the post by saying that zink should be **100%-1000% faster** (not a typo) in most scenarios where it was previously much slower than native GL drivers.

Yeah, Big Triangle knows who we are now.