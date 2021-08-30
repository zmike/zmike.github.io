---
published: false
---
## The Struggle Continues

Everyone's seen the [Phoronix benchmark numbers](https://www.phoronix.com/scan.php?page=article&item=zink-sub-alloc) by now, and though there seems to be a lot of confusion over how to calculate the percentage increase between "game does not run" and "game runs", it seems like a couple people out there at Big Triangle are starting to take us seriously.

With that said, even my parents are asking me what the deal is with this one result in particular:

[![ohno.png](https://openbenchmarking.org/embed.php?i=2108218-PTS-ZINKBENC96&sha=0ba8d3f49d13&p=2)](https://openbenchmarking.org/embed.php?i=2108218-PTS-ZINKBENC96&sha=0ba8d3f49d13&p=2)

Performance isn't supposed to go down. Everyone knows this. The version numbers go up and so does the performance as long as it's not Javascript-based.

Enraged, I sprinted to my computer and searched for **tesseract game**, which gave me the entirely wrong result, but I eventually did manage to find the right one. I fired up zink-wip, certain that this would end up being some bug I'd already fixed.

Unfortunately, this was not the case.

[![tesseract-bad.png]({{site.url}}/assets/tesseract/tesseract-bad.png)]({{site.url}}/assets/tesseract/tesseract-bad.png)

I vowed not to sleep, rebase, leave my office, or even run another application until this was resolved, so you can imagine how pleased I am to be writing this post after spending way too much time getting to the bottom of everything.

## Speculation Interlude
Full disclosure: I didn't actually go and see why performance went down. I'm pretty sure it's just the result of having improved buffer mapping to be better in most cases, which ended up hurting this case.

## But Why
...is the performance so bad?

A quick profiling revealed that this was down to a Gallium component called **vbuf**, used for translating vertex buffers and attributes from the ones specified by the application to ones that drivers can actually support. The component itself is fine, the problem was that, ideally, it's not something you ever want to be hitting when you want performance.

Consider the usual sequence of drawing a frame:
* generate and upload vertex data
* bind some descriptors
* maybe throw in a query or two if you need some spice
* draw
* repeat until frame is done

This is all great and normal, but what would happen—just hypothetically of course—if instead it looked like this:
* generate and upload vertex data
* stall and read vertex data
* rewrite vertex data in another format and reupload
* bind some descriptors
* maybe throw in a query or two if you need some spice
* draw
* repeat until frame is done

Suddenly the driver is now stalling multiple times per frame on top of doing lots of CPU work!

Incidentally, this is (almost certainly) why performance appeared to have regressed: the vertex buffer is now device-local and can't be mapped directly, so it has to be copied to a new buffer before it can be read, which is even slower.

## Just AMD Problems

**DISCLAIMER:** We're going deep into meme territory now, so let's all dial down the seriousness about a thousand notches before posting about how much I hate AMD or whatever.

[![vertexattribmeme.png]({{site.url}}/assets/vertexattribmeme.png)]({{site.url}}/assets/vertexattribmeme.png)

Unlike cool hardware, AMD opts to not support features which might be less performant. I assume this is in the hopes that developers will Make The Right Choice and not use those features, but obviously developers are gonna develop, and so it is that Tesseract-The-Game-But-Not-The-One-On-Steam uses 3-component vertex attributes that aren't supported by AMD hardware, necessitating the use of vbuf to translate them to 4-component attributes that can be safely used.

## Decomposition
