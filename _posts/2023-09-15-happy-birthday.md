---
published: false
---
# Not Mine

But if you're reading, thanks for everything.

# Glamorous

I planned to blog about it a while ago, but then I didn't and news sites have since broken the news: Zink from Mesa main can run finally xservers.

Yes, it's true. For the first time ever, you can install Mesa (from git) and use zink (with environment variables) to run your entire system (unless you're [on Intel](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24700)).

But what was so challenging about getting this to work? The answer won't surprise you.

# WSI
Fans of the blog know that I'm no fan of WSI. If I had my way, GPUs would render to output buffers that we could peruse at our leisure using whatever methods we had at our disposal. Alas, few others share my worldview and so we all must suffer.

The root of all evil when it comes to computers is synchronization. This is triply so for anything GPU-related, and when all this "display server" chicanery is added in the evilness value becomes one of those numbers so large that numerologists are still researching naming possibilities. There are two types of synchronization used with WSI:
* implicit sync - "just fucking do it"
* explicit sync - "I'll tell you exactly when to do it"

From a user perspective, the former is obviously superior since there's less code to manage. The downside is that on the driver side things become more complex, as implicit sync is effectively layered atop explicit sync.

Another way of looking at it is:
* implicit sync - OpenGL
* explicit sync - Vulkan

And, since xservers run on GL, you can see where this is going.