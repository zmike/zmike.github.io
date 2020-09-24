---
published: false
---
## Finally, Performance

For a long time now, I've been writing about various work I've done in the course of getting to GL 4.6. This has generally been feature implementation work, with an occasional side of bug hunting and fixing, and so I haven't been too concerned about performance.

I'm not done with feature work. There's still tons of things missing that I'm planning to work on.

I'm not done with bug hunting or fixing. There's still tons of bugs (like how currently `spec@!opengl 1.1@streaming-texture-leak` ooms my system and crashes all the other tests trying to run in parallel) that I'm going to fix.

But I wanted a break, and I wanted to learn some new parts of the driver instead of just slapping more extensions in.

For the moment, I've been focusing on the Unigine Heaven benchmark since there's tons of room for improvement, though I'm planning to move on from that once I get bored and/or hit a wall. Here's my starting point, which is taken from the patch in my branch with the summary `zink: add batch flag to determine if batch is currently in renderpass`, some 300ish patches ahead of the main branch's tip:

![start.png]({{site.url}}/bench1/start.png)

This is running as `./heaven_x64 -project_name Heaven -data_path ../ -engine_config ../data/heaven_4.0.cfg -system_script heaven/unigine.cpp -sound_app openal -video_app opengl -video_multisample 0 -video_fullscreen 0 -video_mode 3 -extern_define ,RELEASE,LANGUAGE_EN,QUALITY_LOW,TESSELLATION_DISABLED -extern_plugin ,GPUMonitor`, and I'm going to be posting screenshots from roughly the same point in the demo as I progress to gauge progress.

Is this an amazing way to do a benchmark?

No.

Is it a quick way to determine if I'm making things better or worse right now?

Given the size of the gains I'm making, absolutely.

Let's begin.

## Architecture
Now that I've lured everyone in with promises of gains and a screenshot with an fps counter, what I really want to talk about is code.

In order to figure out the most significant performance improvements for zink, it's important to understand the architecture. At the point when I started, zink's batches (C name for an object containing a command buffer and a fence as well as references to all the objects submitted to the queue for lifetime validation) worked like this for draw commands:
* have 4 batches
* each batch has 1 command buffer
* each batch has 1 descriptor pool
* each batch can allocate at most 1000 descriptor sets
* each batch can allocate at most 
* each batch has 1 fence
* each batch, before being reused, waits on its fence and then resets its state
* render passes are started and ended on a given batch as needed
* render passes, when ended, trigger queue submission for the given command buffer
* render passes allocate their descriptor set on-demand just prior to 