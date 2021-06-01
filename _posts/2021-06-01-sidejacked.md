---
published: false
---
## I Hate Construction.

Specifically when it's right outside my house and starts at 5:30AM with heavy machinery moving around.

With this said, I'm overdue for a post, and if I don't set a good example by continuing to blog, why would anyone else?

Let's see what's on the agenda:
* ~~handwaving about C++ draw templates~~
* ~~some obscure vbuf thing~~
* ~~shower~~
* make zink usable for gaming
* improve shader caching

Looks like the next thing on the list is shader caching.

## The Art Of The Cache
If you're a long-time zink connoisseur, or if you're just a casual reader of the blog, you know that zink has a shader cache.

But did you know that it doesn't actually do anything?

Indeed, it was to my chagrin that, upon diving back into my slapdash pipeline cache implementation, I discovered that it was doing absolutely nothing. And this was a different nothing than the time that I didn't actually pass the cache back to the vulkan driver! Yes, this was the nothing of *I have a cache, why am I still compiling a hundred pipelines per frame?* that the occasional lucky developer runs into every now and again.