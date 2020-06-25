---
published: true
---
## A quick word

I didn't budget my time well today, so here's a very brief post about neat features in [piglit](https://gitlab.freedesktop.org/mesa/piglit), the test suite/runner for mesa.

Piglit is my go-to for verifying OpenGL behaviors. It's got tons of fiendishly nitpicky tests for core functionality and extensions, and then it also provides this same quality for shaders with an unlimited number of shader tests.

When working on a new extension or modifying existing behavior, it can be useful to do quick runs of the tests verifying the behavior that's being modified. A full piglit run with Zink takes around 10-20 minutes, which isn't even enough to get in a good rhythm for some desk curls, so it's great that there's functionality for paring down the number of tests being run.

## Regex testing
Piglit provides test inclusion and exclusion support using regular expressions. 
* With `-t`, a regex for tests to run can be specified, e.g., `-t '.*arb_uniform_buffer_object.*'`
* With `-x`, a regex for tests to skip can be specified, e.g., `-x .*glx.*`

These options can be used to cut down runtimes as well as ignore tests with intermittent failures when those results aren't interesting.

## That's it
No, really, I said it would be brief.
