---
published: false
---
## ...For 2021

Zink is done. It's finished. I've said it before, I've said it again after that, and I even said I meant it that one time, but now I'm serious.

Super serious.

We're done here. There's no need to ever check this blog again, and you don't have to update zink ever again if you've pulled in the last week.

## Mesa 21.3
This is it. This is the release where everything is going to change.

Why is that?

Because zink is pretty fast™ now. And it can run lots of stuff.

Blog enthusiasts will recall all that time ago when [zink was over]({{site.url}}/zink-is-over) that I noted a number of Big Name™ games that zink could now run, including **Metro: Last Light Redux** and **HITMAN (Agent 47's Deluxe Psychedelic Trip Edition)**.

True blog connoisseurs will recall [when zink started to pull ahead of real GL drivers in features]({{site.url}}/a-brief-respite) in order to run Warhammer 40k: Dawn of War.

But how many die-hard blog fans are here from the future and can remember when I posted that **Bioshock Infinite** now runs on RADV instead of hanging?

It's hard to overstate the amount of work that's gone into zink for this release. Over 400 patches amounted to ES 3.2, a suballocator, and a slew of random extensions to improve compatibility and performance across the board.

If you find a game or app* which doesn't run on zink 21.3, play the lottery. It's your day.

* Except running it for your Xorg session or Wayland compositor. Don't try this without supervision.

## Bioshock Infinite Now Runs On Zink

[![bioshock.png]({{site.url}}/assets/bioshock.png)]({{site.url}}/assets/bioshock.png)

## Zink+RADV: New BFFs

As part of a cross-training exercise, I've been *hanging around* with the hangmaster himself, the Baron of Black Screens, the Liege of Lost Sessions, Samuel Pitoiset. Together we've (but mostly him while I watch) been stamping out a colossal number of pathological hangs and crashes with zink on RADV.

At present, zink on RADV has only around **200 failures in the GL 4.6 conformance suite**. It's not quite there yet, but considering the number was well over 1000 just a week ago, there's a lot of easy cleanup work that can be done here in both drivers to knock that down further.

## Will 2022 Be The Year Of Zink Conformance?

## What's Next?
Bug fixes.

Lots of bug fixes.

Seriously, there's so, so many bug fixes coming.

I'm planning to take December off entirely, and before then I have some fun surprises to unveil.

For example: any guesses which Vulkan driver I'll be showing zink off on next? Hint: B I G F P S.