---
published: false
---
## Improve Your Pipe!

It's no secret that CPU renderers are slower than GPU renderers. But at the same time, CPU renderers are crucial for things like CI and also not hanging your current session by testing on a live GPU when you're deep into Critical Rewrites.

So I've spent some time today doing some rewrites in the \*pipe section of mesa, and let's just say that the pipe was good with zink before, but it's much, much better now.

Here's where I started on piglit's `drawoverhead` test:
```
   #, Test name                                              ,    Thousands draws/s, Difference vs the 1st
   1, DrawElements ( 1 VBO| 0 UBO|  0    ) w/ no state change,                 1517, 100.0%
   2, DrawElements ( 4 VBO| 0 UBO|  0    ) w/ no state change,                 1519, 100.2%
   3, DrawElements (16 VBO| 0 UBO|  0    ) w/ no state change,                 1112, 73.3%
   4, DrawElements ( 1 VBO| 0 UBO| 16 Tex) w/ no state change,                  524, 34.5%
   5, DrawElements ( 1 VBO| 8 UBO|  8 Tex) w/ no state change,                  583, 38.4%
  ```
  
  
  
  