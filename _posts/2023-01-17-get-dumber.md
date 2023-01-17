---
published: false
---
## Have You Ever...

Had one of those moments when you looked at some code, looked at the output of your program, and then exclaimed COMPUTER, WHY YOU SO DUMB?

Of course not.

Nobody uses the spoken word in the current year.

But you've definitely complained about how dumb your computer is on IRC/discord/etc.

And I'm here today to complain in a blog post.

My computer (compiler) is really fucking dumb.

## HOW DUMB IS IT?

I know you're wondering how dumb a computer (compiler) would have to be to prompt an outraged blog post from me, the paragon of temperance.

Well.

Let's take a look!

> Disclaimer: I have no idea what's happening here, and you are now reading at your own peril.

Some time ago, avid blog followers will recall that I created [vkoverhead](https://github.com/zmike/vkoverhead/) for evaluating CPU overhead in Vulkan drivers. Intel and NVIDIA have both contributed since its inception, which has no relevance but I'm sneaking it in as today's fun fact because nobody will read the middle of this paragraph. Using vkoverhead, I've found something very strange on RADV. Something bizarre. Something terrifying.

**debugoptimized builds of Mesa are faster than release builds for some cases**.

