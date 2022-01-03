---
published: false
---
## We Back

The blog is back. I know everyone's been furiously spamming F5 to see if there were any secret new posts, but no. There were not.

Today's the first day of the new year, so I had to dig deep to remember how to do basic stuff like shitpost on IRC. And then someone told me jekstrand was going to Broadcom to work on Windows network drivers?

I'm just gonna say it now:

**2022 has gone too far.**

I know it's early, I know some people are seeing this as a hot take, but I'm throwing the statement down before things get worse.

Knock it off, 2022.

## Zink
Somehow the driver is still in the tree, still builds, and still runs. It's a miracle.

Thus, since there were obviously no other matters more pressing than not falling behind on MesaMatrix, I spent the morning figuring out how to implement [ARB_sparse_texture](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_sparse_texture.txt).

Was this the best decision when I didn't even remember how to make meson [clear its dependency cache](https://github.com/mesonbuild/meson/issues/6180)? No. No it wasn't.

But [I did it anyway](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/14381) because here at SGC, we take bad ideas and turn them into code.

Your move, 2022.