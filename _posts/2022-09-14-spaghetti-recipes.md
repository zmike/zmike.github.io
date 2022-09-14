---
published: false
---
## Homemade Spaghetti

As I've mentioned in a couple posts, I'm somewhat of a spaghetti expert. In my opinion, the longer and more plentiful the spaghetti is, the better.

So let's take a look at one of my favorite spaghetti recipes.

## Step 1: Read The Label
Today's spaghetti comes from my new favorite brand of spaghetti feed, [vkoverhead](https://github.com/zmike/vkoverhead). It's a simple brand, but it really gets the job done when it comes to growing great spaghetti. This particular spaghetti feed is `vkoverhead -test 0`, which is the most simple type. It grows the kind of spaghetti that everyone notices because it's a staple of all graphics diets.

If I check out the state of this spaghetti feed with RADV now, I see the following:
```
$ ./vkoverhead -test 0 -output-only -duration 3
28345
```

Thus, I can see that I'm getting 28.3 million draws/second. Not too bad. Let's check AMDPRO to get a little competition going.

```
$ VK_ICD_FILENAMES=/home/zmike/amd_pro.json ./vkoverhead -test 0 -output-only -duration 3
32889
```

what

[![mesa.png]({{site.url}}/assets/mesa.png)]({{site.url}}/assets/mesa.png)

What.

[![rotated_mesa.png]({{site.url}}/assets/rotated_mesa.png)]({{site.url}}/assets/rotated_mesa.png)

WHAT.

[![angry_mesa.png]({{site.url}}/assets/angry_mesa.png)]({{site.url}}/assets/angry_mesa.png)

## It's Totally Cool
...that AMDPRO is 15% faster than RADV. Yup, it's totally fine. No problems here, no sir, not with me, not even a little.

Cool as a cucumber over here.

