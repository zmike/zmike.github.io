---
published: false
---
## TFW Long Game Loads

I'm typing this up between loads and runs of various games I'm testing, since bug reports for games are somehow already a thing, and there's a lot of them.

The worst part about testing games is the unbelievably long load times (and startup videos) most of them have, not to mention those long, panning camera shots at the start of the game before gameplay begins and I can start crashing.

But this isn't a post about games.

No, no, there's plenty of time for such things.

This is a combo post: part roundup because blogging has been sporadic the past couple weeks, and part feature.

## The Roundup
The big Mesa 21.1 branchpoint happened yesterday, and I'm pretty pleased with the state of zink in this upcoming release.

**Things you should expect to see:**
* GL 4.6
* ES 3.1
* Reasonable performance in many cases

**Things you should not expect to see:**
* Most (any?) AAA games working; I've kept GL compat contexts clamped to 3.0 in a certainly-futile attempt to cut down on the absolute deluge of bug tickets I'm expecting once everyone tries to run their favorite games/emulators/whathaveyou with the shipped version
  * This functionality remains enabled in zink-wip and will be dumped into mainline soon
* ???
* Tough to say, honestly, since this is effectively a version of zink that is, to me, 5-6 months old with a few other patches sprinkled in here and there

And here's the zink-wip roundup from the past however long since I did the last one:
* I [doubled blending performance](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/10180) in many cases by fixing an incredibly old TODO item regarding using linear tiled images for scanout; this is in the 21.1 release solely to avoid super embarrassing numbers on Phoronix benchmarks. Yeah, I'm talking to you.