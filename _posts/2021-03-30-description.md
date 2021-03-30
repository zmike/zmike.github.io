---
published: false
---
## Yeah, Again

It's been a while since I blogged about descriptors, so let's fix that since this site may as well be called Super Good Descriptors.

Last time, I talked a bit about descriptors 3.0: lazy descriptors. The idea I settled on here was to do templated updates, and to do the absolute minimal amount of work possible for each draw while still never reusing any written-to descriptor sets once they'd been cycled out.

Lazy descriptors worked out great, and I'm sure many of you have grown to enjoy the `ZINK: USING LAZY DESCRIPTORS` log message that got spammed on startup over the past couple months of zink-wip.

Well, times have changed, and this message will no longer fill your terminal by default after today's (20210330) zink-wip snapshot.

## Modes
New today is the `ZINK_DESCRIPTORS` environment variable, supporting three modes of operation:
* `auto` - the default, which attempts to detect system capabilities and use caching with templated updates
* `lazy` - the mode we all know and love from the past few months
* `notemplates` - this is the old-style caching mechanism

`auto` mode moves zink one step closer to my eventual goal, which is to be able to use driconf to do application-based mode changes to disable caching for apps which never reuse resources. Also potentially useful would be the ability to dynamically disable caching on a pipeline-by-pipeline basis while an application is running if too many cache misses are detected.