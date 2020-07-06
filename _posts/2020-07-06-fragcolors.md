---
published: false
---
## Extensions Extensions Extensions

Yes, somehow there are still more extensions left to handle, so the work continues. This time, I'll be talking briefly about a specific part of [EXT_multiview_draw_buffers](https://www.khronos.org/registry/OpenGL/extensions/EXT/EXT_multiview_draw_buffers.txt), namely:

```
If a fragment shader writes to "gl_FragColor", DrawBuffersIndexedEXT
specifies a set of draw buffers into which the color written to
"gl_FragColor" is written. If a fragment shader writes to
gl_FragData, DrawBuffersIndexedEXT specifies a set of draw buffers
into which each of the multiple output colors defined by these
variables are separately written. If a fragment shader writes to
neither gl_FragColor nor gl_FragData, the values of the fragment
colors following shader execution are undefined, and may differ
for each fragment color.
```
The gist of this is:
* if a shader writes to `gl_FragColor`, then the value of `gl_FragColor` is output to all color attachments
* if a shader writes to `gl_FragData[n]`, then this value corresponds to the indexed color attachment

## As expected, this is a problem
`SPIR-V` has no builtin for a color decoration on variables, which means that `gl_FragColor` goes through as a regular variable with no special handling. As such, there's similarly no special handling in the underlying Vulkan driver which splits the output of this variable out to the various color attachments, which means that only the first color attachment will have an expected result when multiple color attachments are present.