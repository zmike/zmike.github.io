---
published: true
---
# New Topic

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

The latter form here is called "lowered" i/o: the derefs for explicit variables have been lowered to intrinsics corresponding to the operation being performed. Such excitement, many detail.

# Change Is Bad

With few exceptions, every mesa driver uses lowered i/o. Zink is one of those exceptions, and the reasons behind it are simple:
* spirv requires explicit variable derefs
* using lowered i/o would take a lot of work
* there's literally no benefit to using lowered i/o

It's a tough choice, but if I had to pick one of these as the "main" reason why I haven't done the move, my response would be yes.

With that said, I'm extremely disgruntled to announce that I have completed the transition to lowered i/o.

Hooray.

The reasoning behind this Sisyphean undertaking which has cost me the past couple weeks along with what shreds of sanity previously remained within this mortal shell are two:
* I've really wanted to delete some [unbelievably gross code](https://gitlab.freedesktop.org/mesa/mesa/-/blob/f71d43ecfb882cd5d777b8a39e0769c40c15b03d/src/gallium/drivers/zink/nir_to_spirv/nir_to_spirv.c#L1704) for [a while](https://gitlab.freedesktop.org/mesa/mesa/-/issues/7045)
* in theory there will someday be this [magical new linker](https://gitlab.freedesktop.org/mesa/mesa/-/issues/8841) that infamous graphics mad scientist Marek Olšák has been working on since the before-times, which will require the use of lowered i/o

It's a tough choice, but if I had to pick one of these as the "main" reason why I have done the move, my response would be yes.

# Before And After
I'll save the details of this for some deep dive posts to pad out my monthly blog counter. For now, let's take a look at the overview: how does this affect "shader stuff" in zink?

The short answer, for that one person who is actively eyeballs-deep in zink shader refactoring, is that it shouldn't have any effect whatsoever. The zink passes that use explicit derefs for i/o are mostly at the end of the compilation chain, and derefs will have been added back in time to avoid needing to touch anything there.

Since this refactor is a tough concept to grasp, I'm providing some flowcharts since it's been far too long since the blog has seen any. This is a basic overview of the zink shader compilation process:

[![](https://mermaid.ink/img/pako:eNpdkUFLAzEQhf_KkKN0KXjMQaEVvdQibqmC8TDtTtdhk-wym12opf_dbGKxOKfwvhd48-ak9m1FSqs-YKAHxlrQFeOt8RDn4-YTiuIOPIuGp1W5gmXrOrYkmUc98W_2TTSgtTy4jCYpsfVmq2FB7GsoHUrIOKqJ9h3LqOH9cXEPb-z3lHGSk2EcbINew1IoBnzhjiz7X1dmV7Z5CrrdrK_5_JJydww0bavhldD-W-UCkzMIo68t9Rp2Mo3xaqYciUOuYlmn6Y9R4YscGaXjs0JpjDL-HH04hLY8-r3SQQaaqaGr_rpV-oC2jypVHFp5zu2nI5x_AOpxgAI?type=png)](https://mermaid.live/edit#pako:eNpdkUFLAzEQhf_KkKN0KXjMQaEVvdQibqmC8TDtTtdhk-wym12opf_dbGKxOKfwvhd48-ak9m1FSqs-YKAHxlrQFeOt8RDn4-YTiuIOPIuGp1W5gmXrOrYkmUc98W_2TTSgtTy4jCYpsfVmq2FB7GsoHUrIOKqJ9h3LqOH9cXEPb-z3lHGSk2EcbINew1IoBnzhjiz7X1dmV7Z5CrrdrK_5_JJydww0bavhldD-W-UCkzMIo68t9Rp2Mo3xaqYciUOuYlmn6Y9R4YscGaXjs0JpjDL-HH04hLY8-r3SQQaaqaGr_rpV-oC2jypVHFp5zu2nI5x_AOpxgAI)

It's a simple process that anyone can understand.

This is the new process side-by-side with the old one for comparison:

[![](https://mermaid.ink/img/pako:eNp9km9P2zAQxr_KyS8nqkh76RdDI-VPtFIq2rENgiInObpTHTu62GEF8d1xbSpQte1eWfd7_Nh67p5FY1sUUgxOOZySWrPqJuPn0kCou0_3MJl8AUMs4Xy2nEFuu540cuKhH_kTmU0QKK3JdwntWpHNVzcSTpDMGpadYpdw6EY69MSjhJ9nJ8fwg0yDB7e1fUSuyO4tpr6rk2RPoozxgdaesbLeVeEblfuN1aiYVK1xkMH6SXHL2-wCPdPgqBmyBast8pDc_m8Q36hVs6mcrfBPr6khF391aRnhzSk7-55_K-bnkM-KxbRYrr7O81O4Pqj03l_M3tMqBigcXI3I8Avd8b8TO71NLPYiHb3eKCMhZwzzXFCPmsxbqIl9kGVxrjer-Uee7Ydabx3ulkPCNSp9MPk9jEoXYjLrGHTNuyqNOBIdcqeoDbv1vLtTihBph6WQ4dgq3pSiNC9Bp7yzy61phHTs8Uj4vn1fRSEflB5CF1tyli_TssadfXkF9sLmGg?type=png)](https://mermaid.live/edit#pako:eNp9km9P2zAQxr_KyS8nqkh76RdDI-VPtFIq2rENgiInObpTHTu62GEF8d1xbSpQte1eWfd7_Nh67p5FY1sUUgxOOZySWrPqJuPn0kCou0_3MJl8AUMs4Xy2nEFuu540cuKhH_kTmU0QKK3JdwntWpHNVzcSTpDMGpadYpdw6EY69MSjhJ9nJ8fwg0yDB7e1fUSuyO4tpr6rk2RPoozxgdaesbLeVeEblfuN1aiYVK1xkMH6SXHL2-wCPdPgqBmyBast8pDc_m8Q36hVs6mcrfBPr6khF391aRnhzSk7-55_K-bnkM-KxbRYrr7O81O4Pqj03l_M3tMqBigcXI3I8Avd8b8TO71NLPYiHb3eKCMhZwzzXFCPmsxbqIl9kGVxrjer-Uee7Ydabx3ulkPCNSp9MPk9jEoXYjLrGHTNuyqNOBIdcqeoDbv1vLtTihBph6WQ4dgq3pSiNC9Bp7yzy61phHTs8Uj4vn1fRSEflB5CF1tyli_TssadfXkF9sLmGg)
