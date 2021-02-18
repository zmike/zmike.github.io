---
published: true
---
## What's Next

It's been a busy week. The CTS fixes and patch drops are coming faster and faster, and progress is swift. Here's a quick note on some things that are on the horizon.

## Features Landing Soon
Zink's in a tough spot right now in master. GL 4.6 is available, but there's still plenty of things that won't work, e.g., running anything at 60fps. These are things I expect (hope) to see land in the repo in the next month or so:
* improved barrier support, which frees up some opportunities with queue refactoring
* removing explicit pre-fencing on every frame (and sometimes multiple times per frame)
* descriptor caching
* various bugfixes which weren't feasible due to architectural issues

All told, just as an example, Unigine Heaven (which can now run in color!) should see roughly a 100% performance improvement (possibly more) once this is in, and I'd expect substantial performance gains across the board.

Will you be able to suddenly play all your favorite GL-based Steam games?

No.

I can't even play all your favorite GL-based Steam games yet, so it's a long ways off for everyone else.

But you'll probably be able to get surprisingly good speed on what things you can run considering the amount of time that will pass between hitting 4.6 and these patchsets merging.

## Features I'm Working On
I spent some time working on Wolfenstein over the past week, but there's some non-zink issues in the way, so that's on the backburner for a while. Instead, I've turned my attention to CTS and begun unloading a dumptruck of resulting fixes into the codebase.

There comes a time when performance is "good enough" for a while, and, after some intense optimizing since the start of the year, that time has come. So now it's back to stabilization mode, and I'm now aiming to have a vaguely decent pass rate in the near term.

Hopefully I'll find some time to post some of the crazy bugs I've been hunting, but maybe not. Time will tell.
