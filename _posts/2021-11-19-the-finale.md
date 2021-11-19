---
published: true
---
## Last Post Of The Year

Yes, we've finally reached that time. It's mid-November, and I've been storing up all this random stuff to unveil now because I'm taking the rest of the year off.

This will be the final SGC post for 2021. As such, it has to be a good one, doesn't it?

## Zink Roundup
It's been a wild year for zink. Does anybody even remember how many times I finished the project? I don't, but it's been at least a couple. Somehow there's still more to do though.

I'll be updating zink-wip one final time later today with the latest Copper snapshot. This is going to be crashier than the usual zink-wip, but that's because zink-wip doesn't have nearly as much cool future-zink stuff as it used to these days. Nearly everything is already merged into mainline, or at least everything that's likely to help with general use, so just use that if you aren't specifically trying to test out Copper.

One of those things that's been a thorn in zink's side for a long time is PBO handling, specifically for unsupported formats like ARGB/ABGR, ALPHA, LUMINANCE, and InTeNsItY. Vulkan has no analogs for any of these, and any app/game which tries to do texture upload or download from them with zink is going to have a very, very bad time, as has been the case with CS:GO, which would take [literal days to reach menus](https://gitlab.freedesktop.org/zmike/mesa/-/issues/89) due to performing fullscreen GL_LUMINANCE texture downloads.

This is now fixed in the course of landing [compute PBO download support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11984), which I [blogged about]({{site.url}}/backish) forever ago since it also yields a 2-10x performance improvement for a number of other cases *in all Gallium drivers*. Or at least the ones that enable it.

CS:GO should now run out of the box in Mesa 22.0, and things like RPCS3 which do a lot of PBO downloading should also see huge improvements.

That's all I've got here for zink, so now it's time once again...

## THIS IS NO LONGER A ZINK BLOG
That's right, it's happening. Change your hats, we're a Gallium blog again for the first time in nearly five months.

Everyone remembers when I promised that you'd be able to [run native Linux D3D9 games on the Nine state tracker]({{site.url}}/different-again). Well, I suppose that's a fancy way of saying Source Engine games, aka the ones Valve ships with native Linux ports, since probably nobody else has shipped any kind of native Linux app that uses the D3D9 API, but still!

That time is now.

Right now.

No more waiting, no new Mesa release required, you can just plug it in and test it out this second for instantly improved performance.

As long as you first acknowledge that **this is not a Valve-official project**, and it's only to be used for educational purposes.

But also, please benchmark it lots and tell me your findings. Again, just for educational purposes. Wink.

## How?
This has been a long time in the making. After the original post, I knew that the goal here was to eventually be able to run these games without needing any kind of specialized Mesa build, since that's annoying and also breaks compatibility with running Nine for other purposes.

Thus I enlisted the further help of Nine expert and image enthusiast, Axel Davy, to help smooth out the rough edges once I was done fingerpainting my way to victory.

The result is [a simple wrapper](https://github.com/zmike/Xnine) which can be preloaded to run any DXVK-compatible (i.e., any of them that support `-vulkan`) Source Engine game on Nine—and obviously this won't work on NVIDIA blob at all so don't bother trying.

In short:
* clone that repo
* right click on `Properties` for e.g., Left 4 Dead 2
* change the command line to `LD_PRELOAD=/path/to/Xnine/nine_sdl.so %command% -vulkan`

For Portal 2 (at present, though this won't always be the case), you'll also need to add `NINE_VHACKS=1` to work around some frogs that were accidentally added to the latest version of the game as a developer-only easter egg.

Then just run the game normally, and if everything went right and you have Nine installed in one of the usual places, you should load up the game with Gallium Nine. More details on that in the repo's README.

## GPU Goes Brrr?
Yes. Very brrr.

Here's your normal GL performance from a simple Portal 2 benchmark:

[![p2-togl.png]({{site.url}}/assets/p2/p2-togl.png)]({{site.url}}/assets/p2/p2-togl.png)

Around 400 FPS.

Here's Gallium Nine:

[![p2-nine.png]({{site.url}}/assets/p2/p2-nine.png)]({{site.url}}/assets/p2/p2-nine.png)

Around 600 FPS.

A 50% improvement with the exact same backend GPU driver isn't too bad for a simple preload shim.

## Can I Get A Side Of SHOTS FIRED With That?
You got it.

What about DXVK?

This isn't an extensive benchmark, but here we go with that too:

[![p2-dxvk.png]({{site.url}}/assets/p2/p2-dxvk.png)]({{site.url}}/assets/p2/p2-dxvk.png)

Also around 600 FPS.

I say "around" here because the variation is quite extreme for both Nine and DXVK based on slight changes in variable clock speeds because I didn't pin them: Nine ranges between 590-610 FPS, and DXVK is 590-620 FPS.

So now there's two solid, open source methods for improving performance in these games over the normal GL version. But what if we go even deeper?

What if we check out some *real* performance numbers?

## Power Consumption
If you've never checked out [PowerTOP](https://01.org/powertop/overview), it's a nice way to get an overview of what's using up system resources and consuming power.

If you've never used it for benchmarking, don't worry, I took care of that too.

Here's some PowerTOP figures for the same Portal 2 timedemo:
* [DXVK]({{site.url}}/assets/p2/dxvk.html)
* [Nine]({{site.url}}/assets/p2/nine.html)

What's interesting here is that DXVK uses 90%+ CPU, while Nine is only using about 25%. This is a gap that's consistent across runs, and it likely explains why a number of people find that DXVK doesn't work on their systems: you still need some amount of CPU to run the actual game calculations, so if you're on older hardware, you might end up using all of your available CPU just on DXVK internals.

## GPU Usage?
Got you covered. Here's a per-second poll (one row per second) from [radeontop](https://github.com/clbr/radeontop).

**DXVK:**

|GPU Usage|VGT Usage|TA Usage|SX Usage|SH Usage|SPI Usage|SC Usage|PA Usage|DB Usage|CB Usage|VRAM Usage|GTT Usage|Memory Clock|Shader Clock|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|35.83%|17.50%|23.33%|28.33%|17.50%|29.17%|28.33%|5.00%|27.50%|26.67%|12.75% 1038.15mb|7.82% 638.53mb|52.19% 0.457ghz|33.52% 0.704ghz|
|35.83%|17.50%|23.33%|28.33%|17.50%|29.17%|28.33%|5.00%|27.50%|26.67%|12.75% 1038.15mb|7.82% 638.53mb|52.19% 0.457ghz|33.52% 0.704ghz|
|36.67%|30.00%|33.33%|35.00%|30.00%|35.00%|32.50%|18.33%|30.83%|28.33%|12.76% 1038.57mb|7.82% 638.53mb|48.88% 0.428ghz|36.95% 0.776ghz|
|75.83%|63.33%|62.50%|66.67%|63.33%|68.33%|65.00%|27.50%|60.83%|53.33%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|95.82% 2.012ghz|
|71.67%|60.00%|60.00%|64.17%|60.00%|66.67%|60.83%|23.33%|56.67%|51.67%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|96.31% 2.023ghz|
|75.00%|62.50%|66.67%|66.67%|62.50%|69.17%|68.33%|23.33%|65.83%|59.17%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|96.71% 2.031ghz|
|63.33%|55.00%|56.67%|58.33%|55.00%|59.17%|59.17%|17.50%|52.50%|50.00%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|89.77% 1.885ghz|
|78.33%|64.17%|64.17%|65.00%|64.17%|69.17%|70.83%|30.00%|63.33%|58.33%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|97.33% 2.044ghz|
|73.33%|60.83%|64.17%|65.00%|60.83%|67.50%|64.17%|29.17%|59.17%|51.67%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|97.39% 2.045ghz|
|60.83%|50.83%|50.00%|53.33%|50.83%|55.00%|50.83%|25.83%|48.33%|45.00%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|95.35% 2.002ghz|
|67.50%|50.00%|55.00%|59.17%|50.00%|60.00%|58.33%|28.33%|52.50%|45.00%|12.76% 1038.73mb|7.82% 638.53mb|100.00% 0.875ghz|87.91% 1.846ghz|

**Nine:**

|GPU Usage|VGT Usage|TA Usage|SX Usage|SH Usage|SPI Usage|SC Usage|PA Usage|DB Usage|CB Usage|VRAM Usage|GTT Usage|Memory Clock|Shader Clock|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|17.50%|11.67%|15.00%|10.83%|11.67%|15.00%|10.83%|3.33%|10.83%|10.00%|7.38% 600.56mb|1.60% 130.48mb|50.38% 0.441ghz|15.76% 0.331ghz|
|17.50%|11.67%|15.00%|10.83%|11.67%|15.00%|10.83%|3.33%|10.83%|10.00%|7.38% 600.56mb|1.60% 130.48mb|50.38% 0.441ghz|15.76% 0.331ghz|
|70.83%|63.33%|65.83%|60.00%|63.33%|68.33%|57.50%|24.17%|56.67%|54.17%|7.35% 598.43mb|1.60% 130.48mb|89.50% 0.783ghz|77.09% 1.619ghz|
|74.17%|70.00%|67.50%|60.00%|70.00%|70.83%|61.67%|17.50%|60.83%|58.33%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|91.03% 1.912ghz|
|78.33%|69.17%|72.50%|65.00%|69.17%|75.83%|65.83%|15.00%|65.83%|64.17%|7.37% 599.80mb|1.60% 130.47mb|100.00% 0.875ghz|93.92% 1.972ghz|
|70.83%|67.50%|64.17%|55.00%|67.50%|67.50%|57.50%|20.83%|55.83%|53.33%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|91.93% 1.930ghz|
|65.00%|64.17%|60.00%|51.67%|64.17%|61.67%|53.33%|18.33%|52.50%|50.83%|7.37% 599.80mb|1.60% 130.47mb|100.00% 0.875ghz|89.95% 1.889ghz|
|74.17%|68.33%|70.00%|60.83%|68.33%|72.50%|65.00%|24.17%|64.17%|58.33%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|92.53% 1.943ghz|
|77.50%|73.33%|73.33%|62.50%|73.33%|75.00%|61.67%|22.50%|62.50%|57.50%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|91.21% 1.915ghz|
|70.00%|65.83%|60.00%|57.50%|65.83%|61.67%|59.17%|24.17%|55.00%|54.17%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|92.69% 1.946ghz|
|70.00%|65.83%|60.00%|57.50%|65.83%|61.67%|59.17%|24.17%|55.00%|54.17%|7.35% 598.42mb|1.60% 130.47mb|100.00% 0.875ghz|92.69% 1.946ghz|

Again, here we see a number of interesting things. DXVK consistently provokes slightly higher clock speeds (because I didn't pin them), which may explain why it skews slightly higher in the benchmark results. DXVK also uses nearly 2x more VRAM and nearly **5x more** GTT. On more modern hardware it's unlikely that this would matter since we all have more GPU memory than we can possibly use in an OpenGL game, but on older hardware—or in cases where memory usage might lead to power consumption that should be avoided because we're running on battery—this could end up being significant.

## Conclusion
Source Engine games run great on Linux. That's what we all care about at the end of the day, isn't it?

But also, if more Source Engine games get ported to DXVK, give them a try with Nine. Or just test the currently ported ones, **Portal 2** and **Left 4 Dead 2**.

I want data.

Lots of data.

Post it here, email it to me, whatever.

## Until 2022
Lots of cool projects still in the works, so stay tuned next year!
