---
published: true
---
## Have You Ever...

Had one of those moments when you looked at some code, looked at the output of your program, and then exclaimed COMPUTER, WHY YOU SO DUMB?

Of course not.

Nobody uses the spoken word in the current year.

But you've definitely complained about how dumb your computer is on IRC/discord/etc.

And I'm here today to complain in a blog post.

My computer (compiler) is really fucking dumb.

## HOW DUMB IS IT?

I know you're wondering how dumb a computer (compiler) would have to be to prompt an outraged blog post from me, the paragon of temperance.

Well.

Let's take a look!

> Disclaimer: You are now reading at your own peril. There is no catharsis to be found ahead.

Some time ago, avid blog followers will recall that I created [vkoverhead](https://github.com/zmike/vkoverhead/) for evaluating CPU overhead in Vulkan drivers. Intel and NVIDIA have both contributed since its inception, which has no relevance but I'm sneaking it in as today's fun fact because nobody will read the middle of this paragraph. Using vkoverhead, I've found something very strange on RADV. Something bizarre. Something terrifying.

**debugoptimized builds of Mesa are (significantly) faster than release builds for some cases**.

Here's an example.

debugoptimized:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  157308,       100.0%
```

release:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  135309,       100.0%
```

[![stupidity-start.png]({{site.url}}/assets/stupidity-start.png)]({{site.url}}/assets/stupidity-start.png)

This is where we are now.

## How?

That's a good question, and for the answer I'm going to turn it over to the blog's new compiler expert, [Mr. Azathoth](https://en.wikipedia.org/wiki/Azathoth):

[![azathoth.png]({{site.url}}/assets/azathoth.png)]({{site.url}}/assets/azathoth.png)

Thanks, champ.

As we can see, I'm about to meddle with dark forces beyond the limits of human comprehension, so anyone who values their sanity/time should escape while they still can.

Let's take a look at a debugoptimized flamegraph to check out what sort of cosmic horror we're delving into.

[![stupid-perf.png]({{site.url}}/assets/stupid-perf.png)]({{site.url}}/assets/stupid-perf.png)

Obviously this is a descriptor updating case, and upon opening up the code for the final function, this is what our eyes will refuse to see:

```c
static ALWAYS_INLINE void
write_image_descriptor(unsigned *dst, unsigned size, VkDescriptorType descriptor_type, const VkDescriptorImageInfo *image_info)
{
   struct radv_image_view *iview = NULL;
   union radv_descriptor *descriptor;

   if (image_info)
      iview = radv_image_view_from_handle(image_info->imageView);

   if (!iview) {
      memset(dst, 0, size);
      return;
   }

   if (descriptor_type == VK_DESCRIPTOR_TYPE_STORAGE_IMAGE) {
      descriptor = &iview->storage_descriptor;
   } else {
      descriptor = &iview->descriptor;
   }
   assert(size > 0);

   memcpy(dst, descriptor, size);
}
```

It seems simple enough. Nothing too terrible here.

## This Is Where It Gets Stupid
In the execution of this test case, the iterated callstack will look something like:

```
radv_UpdateDescriptorSets ->
radv_update_descriptor_sets_impl (inlined) ->
write_image_descriptor_impl (inlined) ->
write_image_descriptor (inlined)
```

That's a lot of inlining.

Typically inlining isn't a problem when used properly. It makes things faster (citation needed) in some cases (citation needed). Note the caveats.

One thing of note in the above code snippet is that `memcpy` is called with a variable `size` parameter. This isn't ideal since it prevents the compiler from making certain assumptions about—What's that, Mr. Azathoth? It's not variable, you say? It's always writing 32 bytes, you say?

```c
case VK_DESCRIPTOR_TYPE_STORAGE_IMAGE:
   write_image_descriptor_impl(device, cmd_buffer, 32, ptr, buffer_list,
                               writeset->descriptorType, writeset->pImageInfo + j);
```

Wow, thanks, Mr. Azathoth, I totally would've missed that!

And *surely* the compiler (GCC 12.2.1) wouldn't fuck this up, right?

[![inline-meme.png]({{site.url}}/assets/inline-meme.png)]({{site.url}}/assets/inline-meme.png)

Which would mean that, hypothetically, if I were to *force* the compiler (GCC 12.2.1) to use the right `memcpy` size then it would have no effect.

**Right?**

```c
static ALWAYS_INLINE void
write_image_descriptor(unsigned *dst, unsigned size, VkDescriptorType descriptor_type, const struct radv_image_view *iview)
{
   if (!iview) {
      memset(dst, 0, size);
      return;
   }

   if (descriptor_type == VK_DESCRIPTOR_TYPE_STORAGE_IMAGE) {
      memcpy(dst, &iview->storage_descriptor, 32);
   } else {
      if (size == 64)
         memcpy(dst, &iview->descriptor, 64);
      else
         memcpy(dst, &iview->descriptor, size);
   }
}
```

```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  141299,       100.0%
```

## That's A Win

Many, many runs of vkoverhead confirm the results, which means that's a solid win.

Right?

Except hold on.

In the course of writing this post, my SSH session to my test machine was terminated, and I had to reconnect. Just for posterity, let's run the same pre/post patch driver through vkoverhead again to be thorough.

pre-patch, release build:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  136943,       100.0%
```

post-patch, release build:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  137810,       100.0%
```

Somehow, with no changes to my environment aside from being in a different SSH session, I've just gained a couple percentage points of performance on the pre-patch run and lost a couple on the post-patch run.

## Second Opinion

It's clear to me that GCC (and computers) can't be trusted. I know what everyone reading this post is now thinking: Why are you still using that antiquated trash compiler instead of the awesome new Clang that does all the things better always?

Well.

I'm sure you can see where this is going.

If I fire up Clang builds of Mesa for debugoptimized and release configurations (same CFLAGS etc), I get these results.

debugoptimized:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  112629,       100.0%
```

release:
```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  132820,       100.0%
```

At least something makes sense there. Now what about a release build with the above changes?

```
$ ./vkoverhead -test 76  -duration 10
vkoverhead running on AMD Radeon RX 5700 XT (RADV NAVI10):
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  127761,       100.0%
```

According to Clang this makes performance worse, but also the performance is just worse overall?!?

## Final Thoughts

Just to see what happens, let's check out AMDPRO:

```
$ VK_ICD_FILENAMES=/home/zmike/amd_icd64.json ./vkoverhead -test 76 -duration 10
vkoverhead running on AMD Radeon RX 5700 XT:
	* descriptor numbers are reported as thousands of operations per second
	* percentages for descriptor cases are relative to 'descriptor_noop'
  76, descriptor_1image,                                  151691,       100.0%
```

I hate computers.