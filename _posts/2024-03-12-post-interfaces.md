# March.

I've had a few things I was going to blog about over the past month, but then news sites picked them up and I lost motivation because there's only so many hours in a day that anyone wants to spend reading things that aren't specification texts. Yeah, that's my life now.

Anyway, a lot's happened, and I'd try to enumerate it all but I've forgotten / lost track / don't care. `git log` me if you're interested. Some highlights:
* damage stuff is in
* RADV supports shader objects so zink can run Tomb Raider (2013) without stuttering
* NVK is about to hit GL conformance on all versions
* I'm working on too many projects to keep track of everything

More on the last one later. Like in a couple months. When I won't get vanned for talking about it.

No, it's not Half Life 3 / Portal 3 / L4D3.

# Interfaces
Today's post was inspired by interfaces: they're the things that make code go brrrrr. Basically Legos, but for adults who never go outside. If you've written code, you've done it using an interface.

Graphics has interfaces too. OpenGL is an interface. Vulkan is an interface.

Mesa has interfaces. It's got some neat ones like Gallium which let you write a whole GL driver without knowing anything about GL. It's got some other ones like the common vk graphics state which let you import all the most cutting edge bugs into your driver that you thought you fixed months earlier.

And then it's got the DRI interfaces. Which, by their mere existence, answer the question "What could possibly be done to make WSI even worse than it already is?"

The DRI interfaces date way back to a time before the blog. A time when now-dinosaurs roamed the earth. A time when Vulkan was but a twinkle in the eye of Mantle, which didn't even exist. I'm talking `Copyright 1998-1999 Precision Insight, Inc., Cedar Park, Texas.` at the top of the file old.

The point of this interface was to let external applications access GL functionality. Specifically the xserver. This was before GLAMOR combined GBM and EGL to enable a better way of doing things that didn't involve brain damage, and it was a necessary evil to enable cross-vendor hardware acceleration using Mesa. Other historical details abound, but this isn't a textbook. The DRI interfaces did their job and enabled hardware-accelerated display servers for decades.

Now, however, they've become cruft. A hassle. A roadblock on the highway to a future where I can run zink on stupid platforms with ease.

# Problem
The first step to admitting there's a problem is having a problem. I think that's how the saying goes, anyway. In Mesa, the problem is any time I (or anyone) want to do something related to the DRI frontend, like [allow NVK users to use zink](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27628), it has to go through DRI. Which means going through the DRI interfaces. Which means untangling a mess of unnecessary function pointers with versioned prototypes meaning they can't be changed without adding a new version of the same function and adding new codepaths which call the new version if available. And guess how many people in the project truly understand how all the layers fit together?

It's a mess. And more than a mess, it's a huge hassle any time a change needs to be made. Not only do the interfaces have to be versioned and changed, someone looking to work on a new or bitrotted platform has to first chase down all the function pointers to see where the hell execution is headed. Even when the function pointers always lead to the same place.

I don't have any memes for this today.

This is my declaration of war.

DRI interfaces: you're officially on notice. I'm [coming for you](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28138).