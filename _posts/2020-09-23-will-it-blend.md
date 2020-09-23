---
published: true
---
## Adventures In Blending

For the past few days, I've been trying to fix a troublesome bug. Specifically, the Unigine Heaven benchmark wasn't drawing most textures in color, and this was hampering my ability to make further claims about zink being the fastest graphics driver in the history of software since it's not very impressive to be posting side-by-side screenshots that look like garbage even if the FPS counter in the corner is higher.

Thus I embarked on adventure.

## First: The Problem
![heaven-pre.png]({{site.url}}/assets/heaven-pre.png)

This was the starting point. The thing ran just fine, but without valid frames drawn, it's hard to call it a test of very much other than how many bugs I can pack into a single frame.

Naturally I assumed that this was going to be some bug in zink's handling of something, whether it was blending, or sampling, or blending and sampling. I set out to figure out exactly how I'd screwed up.

## Second: The Analysis

I had no idea what the problem was. I phoned [Dr. Render](https://renderdoc.org), as we all do when facing issues like this, and I was told that I had problems.

![heaven-renderdoc.png]({{site.url}}/assets/heaven-renderdoc.png)

Lots of problems.

The biggest problem was figuring out how to get anywhere with so many draw calls. Each frame consisted of 3 render passes (with hundreds of draws each) as well as a bunch of extra draws and clears.

There was a lot going on. This was by far the biggest thing I'd had to fix, and it's much more difficult to debug a game-like application than it is a unit test. With that said, and since there's not actually any documentation about "What do I do if some of my frame isn't drawing with color?" for people working on drivers, here were some of the things I looked into:

* Disabling Depth Testing

Just to check. On IRIS, which is my reference for these types of things, the change gave some neat results:

![heaven-nodepth.png]({{site.url}}/assets/heaven-nodepth.png)

How bout that.

On zink, I got the same thing, except there was no color, and it wasn't very interesting.

* Checking sampler resources for depth buffers

On an #executive suggestion, I looked into whether a z/s buffer had snuck into my sampler buffers and was thus providing bogus pixel data.

It hadn't.

* Checking Fragment Shader Outputs

This was a runtime version of my usual shader debugging, wherein I try to isolate the pixels in a region to a specific color based on a conditional, which then lets me determine which path in the shader is broken. To do this, I added a helper function in `ntv`:
```c
static SpvId
clobber(struct ntv_context *ctx)
{
   SpvId type = get_fvec_type(ctx, 32, 4);
   SpvId vals[] = {
      emit_float_const(ctx, 32, 1.0),
      emit_float_const(ctx, 32, 0.0),
      emit_float_const(ctx, 32, 0.0),
      emit_float_const(ctx, 32, 1.0)
    };
   printf("CLOBBERING\n");
   return spirv_builder_emit_composite_construct(&ctx->builder, type, vals, 4);
}
```
This returns a vec4 of the color RED, and I cleverly stuck it at the end of `emit_store_deref()` like so:
```c
if (ctx->stage == MESA_SHADER_FRAGMENT && var->data.location == FRAG_RESULT_DATA0 && match)
   result = clobber(ctx);
```
`match` in this case is set based on this small block at the very start of `ntv`:
```c
if (s->info.stage == MESA_SHADER_FRAGMENT) {
   const char *env = getenv("TEST_SHADER");
   match = env && s->info.name && !strcmp(s->info.name, env);
}
```
Thus, I could set my environment in gdb with e.g., `set env TEST_SHADER=GLSL271` and then zink would swap the output of the fragment shader named `GLSL271` to RED, which let me determine what various shaders were being used for. When I found the shader used for the lamps, things got LIT:

![heaven-lamps.png]({{site.url}}/assets/heaven-lamps.png)

But ultimately, even though I did find the shaders that were being used for the more general material draws, this ended up being another dead end.

* Verifying Blend States

This took me the longest since I had to figure out a way to match up the Dr. Render states to the runtime states that I could see. I eventually settled on adding breakpoints based on index buffer size, as the chart provided by Dr. Render had this in the vertex state, which made things simple.

But alas, zink was doing all the right blending too.

* Complaining

As usual, this was my last resort, but it was also my most powerful weapon that I couldn't abuse too frequently, lest people come to the conclusion that I don't actually know what I'm doing.

Which I do.

And now that I've cleared up any misunderstandings there, I'm not ashamed to say that I went to #intel-3d to complain that Dr. Render wasn't giving me any useful draw output for most of the draws under IRIS. If even zink can get some pixels out of a draw, then a more compliant driver like IRIS shouldn't be having issues here.

I wasn't wrong.

## The Magic Of Dual Blending
It turns out that the Heaven benchmark is buggy and expects the D3D semantics for dual blending, which is why mesa knows this and informs drivers that they need to enable workarounds if they have the need.

As usual, the folks at Intel with their encyclopedic knowledge of how I needed to make my code better were quick to point out the exact problem, which then just left me with the relatively simple tasks of:
* hooking zink up to the driconf build
* checking driconf at startup so zink can get info on these application workarounds/overrides
* adding shader keys for forcing the dual blending workaround
* writing a NIR pass to do the actual work

The result is not that interesting, but here it is anyway:
```c
static bool
lower_dual_blend(nir_shader *shader)
{
   bool progress = false;
   nir_variable *var = nir_find_variable_with_location(shader, nir_var_shader_out, FRAG_RESULT_DATA1);
   if (var) {
      var->data.location = FRAG_RESULT_DATA0;
      var->data.index = 1;
      progress = true;
   }
   nir_shader_preserve_all_metadata(shader);
   return progress;
}
```
In short, D3D expects to blend two inputs based on their locations, but in Vulkan and OpenGL, the blending is based on index. So here, I've just changed the location of `gl_FragData[1]` to match `gl_FragData[0]` and then incremented the index, because [Fragment outputs identified with an Index of zero are directed to the first input of the blending unit associated with the corresponding Location. Outputs identified with an Index of one are directed to the second input of the corresponding blending unit](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#interfaces-fragmentoutput).

And now nice things can be had:

![heaven-post.png]({{site.url}}/assets/heaven-post.png)

Tune in tomorrow when I strap zink to a rocket and begin counting down to blastoff.
