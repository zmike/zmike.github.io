---
published: false
---
## After Some Time

```
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: Collabora Ltd (0x8086)
    Device: zink (Intel(R) Iris(R) Plus Graphics (ICL GT2)) (0x8a52)
    Version: 20.3.0
    Accelerated: yes
    Video memory: 11690MB
    Unified memory: yes
    Preferred profile: core (0x1)
    Max core profile version: 4.6
    Max compat profile version: 3.0
    Max GLES1 profile version: 1.1
    Max GLES[23] profile version: 3.2
OpenGL vendor string: Collabora Ltd
OpenGL renderer string: zink (Intel(R) Iris(R) Plus Graphics (ICL GT2))
OpenGL core profile version string: 4.6 (Core Profile) Mesa 20.3.0-devel (git-b229c29e4d)
OpenGL core profile shading language version string: 4.60
```

It's been about three months, and zink has reached GL 4.6 and GLES 3.2 (conditionally) in my branch. Currently I'm at a 91% pass rate on piglit tests, which seems pretty good considering I'm forcing ANV to do some fp64 stuff despite my GPU not supporting it.