---
published: false
---
## Enhance Your Pipe!

It's no secret that CPU renderers are slower than GPU renderers. But at the same time, CPU renderers are crucial for things like CI and also not hanging your current session by testing on a live GPU when you're deep into Critical Rewrites.

So I've spent some time today doing some rewrites in the \*pipe section of mesa, and let's just say that the pipe was good with zink before, but it's much, much better now.

Here's where I started on piglit's `drawoverhead` test under Lavapipe:
```
   #, Test name                                              ,    Thousands draws/s, Difference vs the 1st
   1, DrawElements ( 1 VBO| 0 UBO|  0    ) w/ no state change,                 1517, 100.0%
   2, DrawElements ( 4 VBO| 0 UBO|  0    ) w/ no state change,                 1519, 100.2%
   3, DrawElements (16 VBO| 0 UBO|  0    ) w/ no state change,                 1112, 73.3%
   4, DrawElements ( 1 VBO| 0 UBO| 16 Tex) w/ no state change,                  524, 34.5%
   5, DrawElements ( 1 VBO| 8 UBO|  8 Tex) w/ no state change,                  583, 38.4%
  ```
  
Nothing too incredible. Modern CPUs with a discrete GPU should be pulling upwards of 15,000k draws/s, and hitting 10% of that is sort of okay.jpg.

## Pipe Faster!
But what if I implemented my Mesa multidraw extensions in Lavapipe?

[![goku.jpg]({{site.url}}/assets/goku.jpg)]({{site.url}}/assets/goku.jpg)

The gist of this extension is that when no other draw parameters have changed except for the starting vertex/index and the number of vertices/indices, an array of those values can just be dumped into the Vulkan driver to optimize out a lot of draw dispatch/validation code, mirroring Marek Olšák's multidraw work in Gallium.

The results, just from doing basic support in Lavapipe without adding any handling in LLVMpipe underneath, speak for themselves:
```
   #, Test name                                              ,    Thousands draws/s, Difference vs the 1st
   1, DrawElements ( 1 VBO| 0 UBO|  0    ) w/ no state change,                 2945, 100.0%
   2, DrawElements ( 4 VBO| 0 UBO|  0    ) w/ no state change,                 2626, 89.2%
   3, DrawElements (16 VBO| 0 UBO|  0    ) w/ no state change,                 1702, 57.8%
   4, DrawElements ( 1 VBO| 0 UBO| 16 Tex) w/ no state change,                 2881, 97.8%
   5, DrawElements ( 1 VBO| 8 UBO|  8 Tex) w/ no state change,                 2888, 98.1%
```

Yup, that's a 100% perf increase.

## Pipe Beyond Space And Time!
But then I thought to myself: What if LLVMpipe could also handle me dumping all this info in?

[![](http://img.youtube.com/vi/kVdlBxdqv8s/0.jpg)](http://www.youtube.com/watch?v=kVdlBxdqv8s)

Indeed, by spending way too much time rewriting and refactoring deep Mesa internals, I have gone beyond:
```
   #, Test name                                              ,    Thousands draws/s, Difference vs the 1st
   1, DrawElements ( 1 VBO| 0 UBO|  0    ) w/ no state change,                 5273, 100.0%
   2, DrawElements ( 4 VBO| 0 UBO|  0    ) w/ no state change,                 4804, 91.1%
   3, DrawElements (16 VBO| 0 UBO|  0    ) w/ no state change,                 2941, 55.8%
   4, DrawElements ( 1 VBO| 0 UBO| 16 Tex) w/ no state change,                 5052, 95.8%
   5, DrawElements ( 1 VBO| 8 UBO|  8 Tex) w/ no state change,                 5057, 95.9%
```

A 3.5x performance boost in `drawoverhead` seemed pretty good, and it also cut my GLES3 CTS runtime by about 20%, so I think I'm done here for now.

## Pipe Fight!
After my recent Swiftshader misadventure, wherein I discovered that it's missing (at least) transform feedback and conditional render extension support, meaning that it can't be used to create a legitimate GL 3.0+ context, esoteric rendering expert Dave Airlie [took benchmarking matters into his own hands](https://airlied.blogspot.com/2021/03/sketchy-vulkan-benchmarks-lavapipe-vs.html) today to battle it out against the other name-brand Vulkan CPU renderer. The results were shocking.
