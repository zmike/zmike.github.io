# Tessellate Your Nightmares

Way, way, way back in the day, there was [EXT_shader_object](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_shader_object.html), and inside Big Triangle there was a lot of debate about how things should work. By this time, I had long since infiltrated their organization. They viewed me as one of their own. Because of this, I was able to influence their sinister operation.

My plotting was even more underhanded than the Big Triangle fat-cats, and I nudged them to design tessellation shader objects in a manner which matched OpenGL. Specifically, all the spacing and vertex ordering mechanics were specified in the same shader stages as OpenGL. The D3D members of Big Triangle were asleep at the wheel. My influence went unopposed.

I was the butterfly flapping my wings to cause a hurricane.

That hurricane manifested years later. Suddenly, those sleeping giants awoke and discovered that shader objects were utterly incompatible with their chosen API. Their howls of rage reverberated across the world.

I was unprepared for the backlash. They wielded their monstrous power and upended everything. Suddenly, spacing and vertex ordering could be specified in *any* tessellation shader stage. It was a nightmare of epic proportions.

# Draw Hell

Lavapipe, unbeknownst to many, is really just llvmpipe wearing a funny hat. This means it inherits all the llvmpipe-isms, including its deepest flaws. One flaw is that llvmpipe is primarily a driver for OpenGL rendering, and most of its internals are structured around that. Chief among them is the tessellation support, which goes through `auxiliary/draw` and then even deeper, into `auxiliary/tessellator`, a mysterious land where few have tread and even fewer have returned.

And all of this code expects tessellation parameters in the shader stages required by OpenGL.

This was fine due to my initial machinations, but it was no longer fine once the more fiendish parts of Big Triangle awoke. The tessellator was broken. Tests were failing. A new crisis had emerged.

# Compatibility

Historically, this mismatch was handled by a function called `merge_tess_info`. It's still present in a number of drivers. And it did work, propagating that info to the right place in lavapipe before being sent to llvmpipe, except for one wrinkle.

Dynamic domain origin.

OpenGL hardcodes this value to lower-left, but in Vulkan it can be either lower-left or upper-left. Lavapipe worked around this by treating upper-left as equivalent to toggling vertex ordering CCW: for the dynamic state, two versions of the shader were compiled, and lavapipe would run the one which corresponded to `(shader_ccw ^ dynamic_domain_ccw)`. This was fine since it was all restricted to the tessellation evaluation shader.

It was no longer fine once vertex ordering could also be specified in tessellation control.

# Another Battle

I had two options: add even more hacks into lavapipe to work around this, or go spelunking deep into gallium to make all the weird bits support setting params in either shader stage. Naturally I went spelunking. This essentially meant shoving `merge_tess_info` into the depths of `auxiliary/draw`.

I won't claim it was pretty or easy, but the battle was won. The forces of good have once again triumphed over the evils of Big Triangle's tessellation monster.
