# After Action Report

The DRIL merge is done, and things are mostly working again after a tumultuous week. To recap, here's everything that went wrong leading up to 24.2-rc1, the reason why it went wrong, and the potential steps that could be taken (but almost certainly won't) to avoid future issues.

# Library Paths
One of the big changes that went in last-minute was a [MR linking all the GL frontend libs to Gallium](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/29771), which is a huge improvement to the old way of using `dlopen` to directly trigger version mismatch errors.

It had some problems, like how it broke Steam. As some readers may have inferred, this was Very Bad, as my employer has some interest in ensuring that Steam does not break.

The core problem in this case has to do with library paths, distro policies, and Steam's own library handling:
* Mesa's libGLX/libEGL/libgbm all link directly to `libgallium.so` now, which means this library must be in the library path
* Traditionally, `libgallium.so` has been installed to `${libdir}/dri`
* I initially suggested installing it to `${libdir}` to avoid library pathing issues, but the criticism I received was that distros would not be friendly towards shipping an unstable library here
* Thus, I came upon the decision to use [rpath](https://en.wikipedia.org/wiki/Rpath) to ensure the `dri` directory was appended to the library path for `libgallium.so`

Unfortunately, there are lots of things that don't fully handle all variations of `rpath`, chief among them Steam. Furthermore, some distros don't use the most optimal implementation of `rpath` (i.e., they use `DT_RPATH` instead of `DT_RUNPATH`), which hits those unimplemented parts of Steam.

The reason(s) this managed to land without issues?
* I was juggling the MR across multiple repos during final testing and CI mashing when I was trying to get it landed, and an intermediate version of the MR landed which updated all the CI `LD_LIBRARY_PATH` variables to include `${libdir}/dri` which I had used for a test run but did not intend to land with the final version
* My test machines all add a lot of extra directories to my `LD_LIBRARY_PATH` to avoid random issues when testing obscure apps

Combined, I wasn't getting adequate testing, so it appeared everything was fine when really nothing was fine.

Lucky for me, Simon McVittie wrote a full [textbook analysis](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11544#note_2499698) of the issue and possible solutions, so this is now fixed.

Ideally in the future I'll have better testing environments and won't be trying to hammer in big MRs minutes before a RC goes out.

# FB Configs Went Missing
DRI is (now) a simple interface that tells Xorg which rendering formats can be used for drawables. This is dependent on the device and driver, but fbconfigs aren't typically something that should vary too much between driver versions. DRIL is meant to split this functionality out of the rest of Mesa so that all the internal interfaces don't have to be a Gordian Knot.

Unfortunately, this means if DRIL has problems determining which formats are usable, the xserver also has problems. There were a lot of problems:
* The original implementation used some pretty suboptimal looping to calculate valid configs (If you're ever using `eglChooseConfigs`, you're probably fucking up) which made it hard to adequately review
* The hardcoded list of valid configs was very limited; basically you got 8/8/8/8 with a couple ZS variants, or you got 10/10/10/2
* No double-buffered configs
* No sRGB variants

This is why there was a sudden deluge of issues about [broken colors](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11546)

[![colors.jpg](https://gitlab.freedesktop.org/-/project/176/uploads/a0a58be7cf8b32fdb40d4315bb3fc159/RIBT7Y5.jpg)](https://gitlab.freedesktop.org/-/project/176/uploads/a0a58be7cf8b32fdb40d4315bb3fc159/RIBT7Y5.jpg)

On my end, I didn't check `glxinfo` output thoroughly enough, nor did I do an exceptionally thorough testing of desktop apps. All the unit tests passed along with CI, which seemed like it should have been enough. Too bad there are no piglit tests which check to see whether various fbconfigs are supported. Maybe I'll write one to ensure there's a CI baseline and catch any future regressions.

# Drivers Stopped Loading
This is a pretty dumb issue, but it was an issue nonetheless: drivers simply stopped loading. This affected any number of embedded (etnaviv) devices and was [fixed](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/30330) by a pretty trivial MR. Also I [broke KMSRO](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11573), which broke even more devices.

Whoops.

The problem here is there's no CI testing, and I have no such devices for testing. Hard to evaluate these types of things when they silently fail.

# But Now We're All Good.
I promise.