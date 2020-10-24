---
published: false
---
## Never Seen Before

A rare Saturday post because I spent so much time this week intending to blog and then somehow not getting around to it. Let's get to the status updates, and then I'm going to dive into the more interesting of the things I worked on over the past few days.

Zink has just hit another big milestone that I've just invented: as of now, my branch is now **passing 97% of piglit tests** up through GL 4.6 and ES 3.2, and a huge improvement from earlier in the week when I was only at around 92%. That's just over 1000 failure cases remaining out of ~41,000 tests. For perspective, a table.

| |IRIS|zink-mainline|zink-wip|
|-|---|---|---|
|**Passed Tests**|43508|21225|40190|
|**Total Tests**|43785|22296|41395|
|**Pass Rate**|99.4%|95.2%|97.1%|

As always, I happen to be running on Intel hardware, so IRIS and ANV are my reference points.

It's important to note here that I'm running piglit tests, and this is very different from CTS; put another way, I may be passing over 97% of the test cases I'm running, but that doesn't mean that zink is *conformant* for any versions of GL or ES, which may not actually be possible at present (without huge amounts of awkward hacks) given the persistent issues zink has with provoking vertex handling. I expect this situation to change in the future through the addition of more Vulkan extensions, but for now I'm just accepting that there's some areas where zink is going to misrender stuff.


