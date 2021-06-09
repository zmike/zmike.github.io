---
published: false
---
## Memes

We've all been there. No matter how 10x someone is or feels, everyone has had a moment where abruptly they say to themselves, **HOW THE FUCK DO THREADS EVEN WORK?**

This may be precipitated by any number of events, including, but not limited to:
* forgetting a lock
* forgetting to unlock
* missing an unlock at an early return
* forgetting to initialize a lock
* forgetting to spawn a thread
* forgetting to signal a conditional
* forgetting to initialize a conditional
* running the test case with the wrong driver

I'm not going to say that I've been there recently.

I'm not going to say that it was today, nor am I going to state, on the record, that at least one existing zink-wip snapshot may or may not be affected by an issue which may or may not be on the above list.

I'm not going to say any of these things.

What I am going to do is talk about a new oom handler I've been working on to handle the dreaded `spec@!opengl 1.1@streaming-texture-leak` case from piglit.

