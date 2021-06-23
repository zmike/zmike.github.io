---
published: true
---
## BREAKING: THIS IS NO LONGER A ZINK BLOG

For today, at least.

Today, this blog is a Gallium blog. And it's a momentous day indeed.

We all know what this is:

[![portal2-title.png]({{site.url}}/assets/portal2-title.png)]({{site.url}}/assets/portal2-title.png)

It's a screenshot of Portal 2 with the Gallium HUD activated and VSync disabled.

But what driver is that underneath?

Well, for today's blog it's RadeonSI, the reference implementation of a Gallium driver.

And **Why is this?**, I can hear you all asking.

What if I told you that this screenshot with 10% higher FPS is also Portal 2 with VSync disabled on RadeonSI using one trick that graphics developers WON'T TELL YOU:

[![portal2-nine-title.png]({{site.url}}/assets/portal2-nine-title.png)]({{site.url}}/assets/portal2-nine-title.png)

Interested?

![](https://media.giphy.com/media/CaiVJuZGvR8HK/giphy.gif)

## Coming Soon (Maybe, And Also Maybe Requiring Some Early 2000s-era Elbow Grease From Anyone Wanting To Try): Source Games On Gallium Nine

We did it.

By assembling an elite team of individuals with a few minutes to spare here and there over the past week, including:
* Josh Ashton, expert spammer of 🐸 emojis
* Axel Davy, expert struct packer
* Me, expert blogger
* Is Such A Thing Even Possible? Why Yes, Yes It Is.

it is now (technically) possible to run DXVK-compatible Source Engine games through Gallium's Nine state tracker, providing a native D3D9 runtime.

Is your Portal 2 in-game FPS sad and barely even 500 like this screenshot?

[![portal2-ingame.png]({{site.url}}/assets/portal2-ingame.png)]({{site.url}}/assets/portal2-ingame.png)

Why not jack it up to more than **TWICE THAT NUMBER**\* with riced out, GPU-fan-shredding technology that Mesa Gallium drivers have been shipping for years?

[![portal2-nine-ingame.png]({{site.url}}/assets/portal2-nine-ingame.png)]({{site.url}}/assets/portal2-nine-ingame.png)

## Disclaimer*
This post does not represent any form of official statement or address from Valve and is only a small project that was started out of boredom while I waited for CTS runs to finish.

This post also does not make any claims or statements regarding performance on other drivers, or performance comparisons using alternative graphics emulation layers, though whew, *it sure would be interesting to see what those kinds of numbers look like*!
