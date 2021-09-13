---
published: false
---
## Zink Is Over: This Time I'm Serious.

Look.

I know what you're gonna say.

"Mike, buddy, c'mon. You *just* said this like a week ago."

That was then, before I set off on my journey to make the rest of [those zany Phoronix benchmark games](https://www.phoronix.com/scan.php?page=article&item=zink-sub-alloc) run instead of crashing or whatever.

What do we got?

## Metro: Last Light Redux

Oh you want some Metro? We got Metro at home.

[![metro.png]({{site.url}}/assets/metro.png)]({{site.url}}/assets/metro.png)

## HITMAN

[![hitman.png]({{site.url}}/assets/hitman.png)]({{site.url}}/assets/hitman.png)

Agent 47, I'm gonna pretend I didn't see that. Pull yourself together.

## Basemark: High Settings

[![basemark.png]({{site.url}}/assets/basemark.png)]({{site.url}}/assets/basemark.png)

It's uh... Mangohud's slowing me down.

## Bioshock Infinite

I bet you're wondering where this one is, huh.

## Warhammer 40,000: Dawn of War

[![dow3-fail.png]({{site.url}}/assets/dow3-fail.png)]({{site.url}}/assets/dow3-fail.png)

Easy as that, ju—Wait, **what**?

This game *requires* bindless textures just to run? Is this a joke? Even fucking DOOM 2016, the final boss of OpenGL, doesn't *require* bindless textures.

Fine.

Totally fine.

Not at all a problem, and I'm sure it'll be easy to do.

Definitely no reason why [only two Mesa drivers total implement it](https://mesamatrix.net/#ExtensionsthatarenotpartofanyOpenGLorOpenGLESversion_Extensions_Extension_GL_ARB_bindless_texture) other than it being some trivial switch that everyone forgot to flip, right?

Probably just a config value here, or maybe a couple lines of code there...

Ignore all the validation errors because descriptor indexing isn't accurately supported...

Add some null checks...

Fire up ASAN to fix a [random stack explosion](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/12829)...

File a [piglit ticket](https://gitlab.freedesktop.org/mesa/piglit/-/issues/57) because two of the unit tests for the extension are bugged...

[![dow3.png]({{site.url}}/assets/dow3.png)]({{site.url}}/assets/dow3.png)

Kapow, first try.

It's just that easy. If you disagree, you are nitpicking and biased.