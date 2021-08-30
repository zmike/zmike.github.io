---
published: false
---
## The Struggle Continues

Everyone's seen the [Phoronix benchmark numbers](https://www.phoronix.com/scan.php?page=article&item=zink-sub-alloc) by now, and though there seems to be a lot of confusion over how to calculate the percentage increase between "game does not run" and "game runs", it seems like a couple people out there at Big Triangle are starting to take us seriously.

With that said, even my parents are asking me what the deal is with this one result in particular:

[![ohno.png](https://openbenchmarking.org/embed.php?i=2108218-PTS-ZINKBENC96&sha=0ba8d3f49d13&p=2)](https://openbenchmarking.org/embed.php?i=2108218-PTS-ZINKBENC96&sha=0ba8d3f49d13&p=2)

Performance isn't supposed to go down. Everyone knows this. The version numbers go up and so does the performance as long as it's not Javascript-based.

So what's going on here?

## Speculation Interlude
Full disclosure: I didn't actually go and see why performance went down. I'm pretty sure it's just the result of having improved buffer mapping to be better in most cases, which ended up hurting this case.

## But Why
...is the performance so bad?

This all comes down to a Gallium component called **vbuf**, used for translating vertex buffers and attributes to ones that drivers can support