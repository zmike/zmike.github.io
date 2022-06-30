---
published: false
---
## Addressing Concerns

After my last post, I received a very real, nonzero number of DMs bearing *implications*.

That's right.

Messages like, "Nice blog post. I'm sure you totally know how to use RenderDoc." and "Nice RenderDoc tutorial." as well as "You don't know how to use RenderDoc, do you?"

I even got a message from Baldur "Dr. Render" Karlsson asking to see my RenderDoc operator's license. Which I definitely have. And it's not expired.

I just, uh, don't have it on me right now, officer.

But we can work something out, right?

Right?

## Community Service

So I'm writing this post of my own free will, and in it we're going to take a look at [a real bug](https://gitlab.freedesktop.org/mesa/mesa/-/issues/6240). Using RenderDoc.

Like a tutorial, almost, but less useful.

If you want a real tutorial, you should be contacting Danylo Piliaiev, Professor Emeritus at Render University, for his self-help guide. Or watch his [free zen meditation tutorial](https://www.youtube.com/watch?v=T2JQq1cxB1o&hd=1). Powerful stuff.

First, let's open up the renderdoc capture for this bug:

[![app.png]({{site.url}}/assets/renderdoc/app.png)]({{site.url}}/assets/renderdoc/app.png)

What's that? I skipped the part where I was supposed to describe how to get a capture?

Fine, fine, let's go back.

## RenderDoc + Zink: Not A HOWTO
If you're not already a maestro of the most powerful graphics debugging tool on the planet, the process for capturing a frame on zink goes something like this:

```
LD_PRELOAD=/usr/lib64/librenderdoc.so MESA_LOADER_DRIVER_OVERRIDE=zink <executable>
```

I've been advised by my lawyer to state for the record that preloading the library in this manner is Not Officially Supported, and the GUI should be used whenever possible. But this isn't a tutorial, so you can read the RenderDoc documentation to set up the GUI capture.

Moving along, if the case in question is a single frame apitrace, as is the case for this bug, there's some additional legwork required:

```
LD_PRELOAD=/usr/lib64/librenderdoc.so MESA_LOADER_DRIVER_OVERRIDE=zink glretrace --loop portal2.trace
```

In particular here, the `--loop` parameter will replay the frame infinitely, enabling capture. There's also the need to use F11 to cycle through until the Vulkan window is selected. And also possibly disable GL support in RenderDoc so it doesn't get confused.

But this isn't a tutorial, so I'm gonna assume that once the trace starts playing at this point, it's easy enough for anyone following along to press F11 a couple times to select Vulkan and then press F12 to capture the frame.

## The Interface

This is more or less what RenderDoc will look like once the trace is opened:

[![app.png]({{site.url}}/assets/renderdoc/app.png)]({{site.url}}/assets/renderdoc/app.png)

Assuming, of course, that you are in one of these scenarios:
* not running on ANV
* running on ANV with [this MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/17309) applied so the app doesn't crash
* not running on Lavapipe since, obviously, this bug doesn't exist there (R E F E R E N C E)

