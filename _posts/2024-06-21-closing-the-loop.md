# You Would Not Believe This Month

I don't have a lot of time. There's a gun to my head. Literally.

John Eldenring is here, and he has a gun pointed at my temple, and he's telling me that if I don't start playing his new downloadable content *now*, I won't be around to make any more posts.

# The Game
Today's game is [Blazblue Centralfiction](https://store.steampowered.com/app/586140/BlazBlue_Centralfiction/). I don't know what this game is, I don't know what it's about, I don't even know what genre it is, but it's got problems.

What kinds of problems?

This is a DX game that embeds video files. In proton, GStreamer is used to play the video files while DXVK goes brrrr in the background. GStreamer uses GL. What do you think this looks like under perf?

[![readpixoof.png]({{site.url}}/assets/readpixoof.png)]({{site.url}}/assets/readpixoof.png)

# The Problem
GStreameur here does the classic `memcpy PIXEL_PACK_BUFFER texture upload -> glReadPixels memcpy download` in order to transcode the video files into something renderable. I say classic because this is a footgun on both ends:
* each upload is full-frame (huge)
* each download is full-frame (huge)

This results in blocking at both ends of the transcode pipeline. A better choice, for Mesa's current state of optimization, would've been to do `glTexImage -> glGetTexImage`. This would leverage all the work that I did however many years ago in [this post]({{site.url}}/backish/) for PBO downloads using compute shaders.

Still, this is the future, so Mesa must adapt. With a few copy/pasted lines and a sprinkle of magical SGC dust (massive compute-based PBO shaders), the flamegraph becomes:

[![readpixyay.png]({{site.url}}/assets/readpixyay.png)]({{site.url}}/assets/readpixyay.png)

Flamegraphs aren't everything though. This has more obvious real world results just from trace replay times:

```
# before
$ time glretrace -b BlazBlue.Centralfiction.trace
/home/zmike/.local/share/Steam/steamapps/common/Proton - Experimental/files/bin/wine-preloader
Rendered 0 frames in 15.4088 secs, average of 0 fps
glretrace -b BlazBlue.Centralfiction.trace  13.88s user 0.35s system 91% cpu 15.514 total

# after
$ time glretrace -b BlazBlue.Centralfiction.trace
/home/zmike/.local/share/Steam/steamapps/common/Proton - Experimental/files/bin/wine-preloader
Rendered 0 frames in 10.6251 secs, average of 0 fps
glretrace -b BlazBlue.Centralfiction.trace  9.83s user 0.42s system 95% cpu 10.747 total
```

Considering this trace only captured the first 4-5 seconds of a 98 second movie, I'd say that's damn good.

Check out [the MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/29841) if you want to test.