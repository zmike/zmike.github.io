---
published: false
---
## A New Look

As every one of my big brained readers knows, zink runs on top of vulkan. As you also know, vulkan uses spirv for its shaders. This means, in general, compiler-y stuff in zink tries to stay as close to spirv mechanics as possible.

Let's look at an example. Here's a very simple fragment shader from glxgears before it undergoes spirv translation:

```
shader: MESA_SHADER_FRAGMENT
source_sha1: {0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x00000000}
name: ff-fs
internal: true
stage: 4
next_stage: 0
inputs_read: 1
outputs_written: 4-11
system_values_read: 0x00000000'00100000'00000000
subgroup_size: 1
first_ubo_is_default_ubo: true
separate_shader: true
flrp_lowered: true
inputs: 1
outputs: 8
uniforms: 0
decl_var shader_in INTERP_MODE_NONE vec4 VARYING_SLOT_COL0 (VARYING_SLOT_COL0.xyzw, 0, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[0] (FRAG_RESULT_DATA0.xyzw, 0, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[1] (FRAG_RESULT_DATA1.xyzw, 1, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[2] (FRAG_RESULT_DATA2.xyzw, 2, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[3] (FRAG_RESULT_DATA3.xyzw, 3, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[4] (FRAG_RESULT_DATA4.xyzw, 4, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[5] (FRAG_RESULT_DATA5.xyzw, 5, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[6] (FRAG_RESULT_DATA6.xyzw, 6, 0)
decl_var shader_out INTERP_MODE_NONE vec4 gl_FragData[7] (FRAG_RESULT_DATA7.xyzw, 7, 0)
decl_var push_const INTERP_MODE_NONE struct gfx_pushconst
decl_function main (0 params)

impl main {
    block b0:   // preds: 
    32     %0 = deref_var &VARYING_SLOT_COL0 (shader_in vec4)
    32x4   %1 = @load_deref (%0) (access=0)
    32     %2 = deref_var &gl_FragData[0] (shader_out vec4)
                @store_deref (%2, %1) (wrmask=xyzw, access=0)
    32     %3 = deref_var &gl_FragData[1] (shader_out vec4)
                @store_deref (%3, %1) (wrmask=xyzw, access=0)
    32     %4 = deref_var &gl_FragData[2] (shader_out vec4)
                @store_deref (%4, %1) (wrmask=xyzw, access=0)
    32     %5 = deref_var &gl_FragData[3] (shader_out vec4)
                @store_deref (%5, %1) (wrmask=xyzw, access=0)
    32     %6 = deref_var &gl_FragData[4] (shader_out vec4)
                @store_deref (%6, %1) (wrmask=xyzw, access=0)
    32     %7 = deref_var &gl_FragData[5] (shader_out vec4)
                @store_deref (%7, %1) (wrmask=xyzw, access=0)
    32     %8 = deref_var &gl_FragData[6] (shader_out vec4)
                @store_deref (%8, %1) (wrmask=xyzw, access=0)
    32     %9 = deref_var &gl_FragData[7] (shader_out vec4)
                @store_deref (%9, %1) (wrmask=xyzw, access=0)
                // succs: b1 
    block b1:
}
```

Notice all the variables and derefs. This is in contrast to what shaders from more hardware-y drivers look like:

```
shader: MESA_SHADER_FRAGMENT
source_sha1: {0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x00000000}
name: ff-fs
internal: true
stage: 4
next_stage: 0
inputs_read: 1
outputs_written: 2
subgroup_size: 1
first_ubo_is_default_ubo: true
separate_shader: true
flrp_lowered: true
inputs: 1
outputs: 1
uniforms: 0
decl_var shader_in INTERP_MODE_NONE vec4 VARYING_SLOT_COL0 (VARYING_SLOT_COL0.xyzw, 0, 0)
decl_var shader_out INTERP_MODE_NONE vec4 FRAG_RESULT_COLOR (FRAG_RESULT_COLOR.xyzw, 4, 0)
decl_function main (0 params)

impl main {
    block b0:  // preds: 
    32    %3 = undefined
    32    %0 = deref_var &VARYING_SLOT_COL0 (shader_in vec4)
    32x4  %1 = @load_deref (%0) (access=0)
    32    %2 = deref_var &FRAG_RESULT_COLOR (shader_out vec4)
    32    %4 = load_const (0x00000000)
               @store_output (%1, %4 (0x0)) (base=4, wrmask=xyzw, component=0, src_type=float32, io location=FRAG_RESULT_COLOR slots=1, xfb(), xfb2())  // FRAG_RESULT_COLOR
               // succs: b1 
    block b1:
}
```

The latter form here is called "lowered" i/o: the derefs for explicit variables have been lowered to intrinsics corresponding to the operation being performed.