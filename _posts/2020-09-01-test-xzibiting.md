---
published: true
---
## Benchmarking vs Testing

Just a quick post for today to talk about a small project I undertook this morning.

As I've talked about repeatedly, I run a lot of tests.

It's practically all I do when I'm not dumping my random wip code by the truckload into the repo.

The problem with this approach is that it doesn't leave much time for performance improvements. How can I possibly have time to run benchmarks if I'm already spending all my time running tests?

## Motivation
As one of the greatest bench markers of all time once said, [**If you want to turn a vision into reality, you have to give 100% and never stop believing in your dream**](https://www.goodreads.com/quotes/1038255-if-you-want-to-turn-a-vision-into-reality-you).

Thus I decided to look for those gains while I ran tests.

Piglit already possesses facilities for providing HTML summaries of all tests (sidebar: the results that I posted last week are actually a bit misleading since they included a number of regressions that have since been resolved, but I forgot to update the result set). This functionality includes providing the elapsed time for each test. It has not, however, included any way of displaying changes in time between result sets.

Until now.

## The Future Is Here
With [my latest MR](https://gitlab.freedesktop.org/mesa/piglit/-/merge_requests/375), the HTML view can show off those big gains (or losses) that have been quietly accumulating through all these high volume sets of unit tests:

![piglit-timing.png]({{site.url}}/assets/piglit-timing.png)

Is it more generally useful to other drivers?

Maybe? It'll pick up any unexpected performance regressions (inverse gains) too, and it doesn't need any extra work to display, so it's an easy check, at the least.
