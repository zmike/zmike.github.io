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

But hwhy? Who would do such a thing?

[![spideymeme.jpg]({{site.url}}/assets/spideymeme.jpg)]({{site.url}}/assets/spideymeme.jpg)

Past recriminations aside, how does a shader/pipeline cache work, anyway? The gist of it in most Mesa drivers is this:

[![](https://mermaid.ink/img/eyJjb2RlIjoic3RhdGVEaWFncmFtLXYyXG5zMTogSGF2ZSBzaGFkZXIgZnJvbSBhcHBcbnMyOiBTZXJpYWxpemUgdG8gTklSIHRleHRcbnMzOiBDb21wdXRlIFNIQTEgaGFzaFxuczQ6IFVzZSBoYXNoIGZvciBjYWNoZSBsb29rdXBcbnM1OiBDYWNoZSBoaXRcbnM3OiBDYWNoZSBtaXNzXG5zNjogSGF2ZSBjb21waWxlZCBzaGFkZXJcbnM4OiBDb21waWxlIG5ldyBzaGFkZXJcbiAgICBbKl0gLS0-IHMxXG4gICAgczEgLS0-IHMyXG4gICAgczIgLS0-IHMzXG4gICAgczMgLS0-IHM0XG4gICAgczQgLS0-IHM1XG4gICAgczQgLS0-IHM3XG4gICAgczUgLS0-IHM2XG4gICAgczcgLS0-IHM4XG4gICAgczggLS0-IHM2IiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQifSwidXBkYXRlRWRpdG9yIjpmYWxzZX0)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic3RhdGVEaWFncmFtLXYyXG5zMTogSGF2ZSBzaGFkZXIgZnJvbSBhcHBcbnMyOiBTZXJpYWxpemUgdG8gTklSIHRleHRcbnMzOiBDb21wdXRlIFNIQTEgaGFzaFxuczQ6IFVzZSBoYXNoIGZvciBjYWNoZSBsb29rdXBcbnM1OiBDYWNoZSBoaXRcbnM3OiBDYWNoZSBtaXNzXG5zNjogSGF2ZSBjb21waWxlZCBzaGFkZXJcbnM4OiBDb21waWxlIG5ldyBzaGFkZXJcbiAgICBbKl0gLS0-IHMxXG4gICAgczEgLS0-IHMyXG4gICAgczIgLS0-IHMzXG4gICAgczMgLS0-IHM0XG4gICAgczQgLS0-IHM1XG4gICAgczQgLS0-IHM3XG4gICAgczUgLS0-IHM2XG4gICAgczcgLS0-IHM4XG4gICAgczggLS0-IHM2IiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQifSwidXBkYXRlRWRpdG9yIjpmYWxzZX0)

Thus a shader gets cached based on its text representation, enabling matching shaders across programs to use the same cache entry. After noting the success of Steam's fossilize-based single file cache, I decided to use a single file for zink's shader cache.

Oops.

The problem in this case was that I was just jamming all the pipelines into a single file, written once at program exit, and expecting the Vulkan driver to figure things out.

But what if the program didn't exit cleanly? Or what if the write failed for some reason?

In short, the pipeline cache was mostly being written as a big block of garbage data. Not very useful.