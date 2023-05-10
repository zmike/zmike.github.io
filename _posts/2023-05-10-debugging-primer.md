---
published: false
---
## Release Pending

If nothing goes wrong, Mesa 23.1 will ship in the next few hours, which means that at last everyone will have a new zink release.

And it's a [big one]({{site.url}}/branched/).

Since I'm expecting lots of people will be testing zink for the first time now that it should be usable for most things, I thought it would be useful to have a post about debugging zink. Or maybe not debugging but issue reporting. That sounds right.

## How2Debug
Zink is a complex driver. It has many components:
* DRI frontend
* Mesa's GL API
* Gallium state tracker
* zink driver
* ntv compiler
* Vulkan driver underneath

There are many systems at play when zink is running, any malfunctions in any of them may lead to bugs. If you encounter a bug, there are a number of steps you can take to try mitigating its effect or diagnosing it.

Let's dig in.

## Step 1: Threads
Zink tries to use a lot of threads. Sometimes they can cause problems. One of the first steps I take when I encounter an issue is to disable them:

`mesa_glthread=false GALLIUM_THREAD=0 ZINK_DEBUG=flushsync MESA_LOADER_DRIVER_OVERRIDE=zink <command>`

* `mesa_glthread=false` disables glthread, which very occasionally causes issues related to vertex buffers
* `GALLIUM_THREAD=0` disables threaded context, which very occasionally causes issues in everything
* `ZINK_DEBUG=flushsync` disables threaded queue submission, which has historically never been an issue but may remove some timing discrepancies to enable bugs to shine through more readily

If none of the above affects your problem, it's time to move on to the next step.

## Step 2: Optimizations
Zink tries a lot of optimizations. Sometimes they can cause problems. One of the first steps I take when I encounter an issue that isn't fixed by the above is to disable them:

`mesa_glthread=false GALLIUM_THREAD=0 ZINK_DEBUG=flushsync,noreorder,norp MESA_LOADER_DRIVER_OVERRIDE=zink <command>`

* `noreorder` disables command reordering, which will probably fix any issue
* `norp` disables renderpass optimization, which is only enabled by default on tiling GPUs so it's definitely not affecting anyone reading this

If none of the above affects your problem, then ~~you're probably screwed~~it's time to move on to the next step.

## Step 3: Synchronization
Zink tries to obey Vulkan synchronization rules. Sometimes these rules are difficult to understand and confusing to implement. One of the first steps I take when I encounter an issue that isn't fixed by the above is to ~~cry a lot~~check out ~~[this blog post](https://themaister.net/blog/2019/08/14/yet-another-blog-explaining-vulkan-synchronization/)~~the synchronization. Manually inspecting this is hard, so there's only a big hammer:

`mesa_glthread=false GALLIUM_THREAD=0 ZINK_DEBUG=flushsync,noreorder,norp,sync MESA_LOADER_DRIVER_OVERRIDE=zink <command>`

* `sync` forces full memory synchronization before every command, which will solve the problem of extant FPS

If none of the above affects your problem, then I totally have more ideas, and I'm gonna blog about them right here, but I gotta go, um, get something. From my...refrigerator. It's just down the street I'll berightback.

## Step 4: Pray
Zink tries to conform to Vulkan API rules. Sometimes these rules are obfuscated by a trillion Valid Usage statements that nobody has time to read or the mental capacity to comprehend in their totality.

And when a problem appears that cannot be mitigated with any of the above, we must pray to our lord and savior, the [Vulkan Validation Layer](https://github.com/KhronosGroup/Vulkan-ValidationLayers/) for guidance:

`mesa_glthread=false GALLIUM_THREAD=0 ZINK_DEBUG=flushsync,noreorder,norp,sync,validation MESA_LOADER_DRIVER_OVERRIDE=zink <command>`

* `validation` enables VVL at runtime, which will (hopefully) explain how I fucked up

If this doesn't fix your problem,

## Step 5: Acceptance
You should still file a ticket if you find an issue. But if any of the above provided some additional info, providing that can reduce the time it takes to resolve that issue.