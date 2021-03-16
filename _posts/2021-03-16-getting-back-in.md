---
published: false
---
## Superposition

I got a [weird bug report](https://gitlab.freedesktop.org/zmike/mesa/-/issues/71) the other day. Apparently Unigine Superposition, the final boss of Unigine benchmarks, was broken in zink(-wip). And it was a regression, which meant that at some point it had worked, but because testing is hard, it then stopped working for a while.

## The Problem
I had no idea what the problem was other than a vague impression it might be queue-related given the perf output in the bug report. The first step here was to figure out what I was dealing with.