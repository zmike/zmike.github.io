# Hm

I'm no stranger to big tree-wide refactors. I've done lots of them. They suck for everyone.

That's why this time I decided to hand over this job to the new high-tech calculator, Claude.

We've had a lot of discussion about AI use in mesa, and while I can appreciate the various factions, I came out sort of vaguely in favor of allowing AI as long as it was used with some discrimination (i.e., a calculator exists to decrease the workload of humans, not increase it). But I had never actually used it other than asking the free models to find me spec references for various things.

Today, that changed.

# Process
I don't know if anyone will find this useful/interesting, but I'm going to document the process I used today.

## Install Claude Code VSCode plugin
It's just the normal one from Anthropic. I linked it to my Claude subscription, where I have a 7-day free trial for Claude Pro activated.

## Prompt
My exact prompt was as follows:

```
in the mesa repo I have open:
* do not create or alter any git commits
* delete support for big endian
```

This took a long while to execute, and it printed out lots of in-progress stuff it was doing. Some neat things I observed along the way were how it refused to modify `src/gallium/drivers/asahi` due to in-tree policy documentation and how it wrote a script to mimic `unifdef` when it was unable to find my `kernel-devel` install directory. One of the times it ran the homegrown script resulted in a fuckup related to line-wrapping, which it detected and then fixed. Cool.

When it was finally finished, it started a test build to verify. The build passed.

## Organize
I glanced over all the changes, and they seemed reasonable. My next prompt:

```
organize the big endian removal into logical git commits:
* each commit must be independently buildable
* each commit must "make sense"
* add `Assisted-by: Claude Code (Opus 5.5 Medium)` at the end of every commit log
```

This also took a while, but it ended up with 15 commits and refused to write commit logs for them because apparently that's against our AI policy. It then tried to kick off builds to verify the bisectability.

Unfortunately, my main workstation is still an ICL laptop from 2019, so this would've taken literal hours. I told Claude it could ssh to my big test machine with a threadripper for this step. It worked.

## Document
Drunk with power, I asked Claude:

```
briefly describe the effects of each commit
```

Then I confirmed it did actually reflect the contents of each commit, massaged the output a bit, and that became the commit logs.

## Fixups
Our AI policy allows individual drivers/components to ban AI-assisted changes. For this reason, `pipe_cap::endianness` could not be removed by Claude. I did this manually. It didn't take long.

## Done?

In total, this took maybe an hour for me, an actual organic human, to get big endian completely removed from mesa. Doing it manually would've taken days, so I think this is a win.

[https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/44863](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/44863)
