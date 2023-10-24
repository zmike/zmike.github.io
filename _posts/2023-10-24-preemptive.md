---
published: true
---
# Your Bug Has Already Been Solved

After yesterday's post, I'm sure my thousands of readers stampeded to install the latest zink and run their system with it, and I salute you for your hard work in finding all those new ways to crash your systems.

Some of those crashes, however, are not my bugs. They're system bugs.

In particular, any of you still using Xorg instead of Wayland will want to create this file:

```
$ cat /etc/X11/xorg.conf.d/30-dmabuf.conf
Section "ServerFlags"
	Option "Debug" "dmabuf_capable"
EndSection
```

This makes your xserver dmabuf-capable, which will be more successful when running things with zink.

Another problem you're likely to have is this console error:

```
DRI3 not available
failed to load driver: zink
```

Specifically you're likely to have this on AMD hardware, and the cause is almost certainly that you've installed some footgun package with a naming variation on `xf86-video-amdgpu`.

Delete this package.

Just delete it. I don't know why distros still make it available, but if you have it installed then you're just footgunning yourself.

If you're still having problems after checking for both of these issues, try turning your computer on.
