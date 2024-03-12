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

And then it's got the DRI interfaces. Which let you simulate the 