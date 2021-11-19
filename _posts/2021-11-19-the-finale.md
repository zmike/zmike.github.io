---
published: false
---
## Last Post Of The Year

Yes, we've finally reached that time. It's mid-November, and I've been storing up all this random stuff to unveil now because I'm taking the rest of the year off.

This will be the final SGC post for 2021. As such, it has to be a good one, doesn't it?

## Zink Roundup
This has been a wild year for zink. Does anybody even remember how many times I finished the project? I don't, but it's been at least a couple. Somehow there's still more to do though.

One of those things that's been a thorn in zink's side for a long time is PBO handling, specifically for unsupported formats like ARGB/ABGR, ALPHA, LUMINANCE, and InTeNsItY. Vulkan has no analogs for any of these, and any app/game which tries to do texture upload or download from them with zink is going to have a very, very bad time, as has been the case with CS:GO, which would take [literal days to reach menus](https://gitlab.freedesktop.org/zmike/mesa/-/issues/89) due to performing fullscreen geometry GL_ALPHA texture downloads.

This is now fixed in the course of landing [compute PBO download support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11984), which I [blogged about]({{site.url}}/backish) forever ago since it also yields a 2-10x performance improvement for a number of other cases *in all Gallium drivers*. Or at least the ones that enable it.

CS:GO should now run out of the box in Mesa 22.0, and things like PCSX3 which do a lot of PBO downloading should also see huge improvements.

That's all I've got here for zink, so now it's time once again...

##THIS IS NO LONGER A ZINK BLOG
That's right, it's happening. Change your hats, we're a Gallium blog again for the first time in nearly five months.

Everyone remembers when I promised that you'd be able to [run native Linux D3D9 games out of the box]({{site.url}}/different-again) on the Nine state tracker. Well, I suppose that's a fancy way of saying Source Engine games, aka the ones Valve ships with native Linux ports, since probably nobody else has shipped any kind of native Linux app that uses the D3D9 API.

That time is now.

Right now.

No more waiting, no new Mesa release required, you can just plug it in and test it out this second for instantly improved performance.

## How?
This has been a long time in the making. After the original post, I knew that the goal here was to eventually be able to run these games without needing any kind of specialized Mesa build, since that's annoying and also breaks compatibility with running Nine for other purposes.

Thus I enlisted the further help of Nine expert and image enthusiast, Axel Davy to help smooth out the rough edges once I was done fingerpainting around the edges.