# Super Fast

Just a quick post to let everyone know that I have clicked merge on [the vroom MR](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28580). Once it lands, you can test the added performance gains with `ZINK_DEBUG=ioopt`.

I'll be enabling this by default in the next month or so once a new GL CTS release happens that fixes all the hundreds of broken tests which would otherwise regress. With that said, I've tested it on a number of games and benchmarks, and everything works as expected.

Have fun.