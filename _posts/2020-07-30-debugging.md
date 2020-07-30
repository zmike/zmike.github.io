---
published: true
---
## The Hidden Dangers of State Tracking

It's another hot one, so let's cool down by checking out a neat bug I came across.

As I've mentioned previously, zink runs a NIR pass to inject a `gl_PointSize` value into the last vertex processing stage of the pipeline for point draws. This works in three parts:
* the NIR pass to add the instructions
* adding the pointsize data into the shader parameters
* the shader parameters get uploaded to the gpu as a constant buffer

Recently, I found a case where the constant buffer for this data was missing, causing tests to crash. I immediately strapped on my snorkel and dove in.

## Is The NIR Pass Running?
Yes, but since it's in a geometry shader, some [recent work](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/5885/diffs?commit_id=67e541dfcb19fc1ebbd17eb18110c8d25c725bbd) is needed.

## Is The Data Being Added Into Shader Parameters?
Again checking out the MR where I added this, I examined the code:
```c
            if (key->lower_point_size) {
               static const gl_state_index16 point_size_state[STATE_LENGTH] =
                  { STATE_INTERNAL, STATE_POINT_SIZE_CLAMPED, 0 };
               _mesa_add_state_reference(params, point_size_state);
               NIR_PASS_V(state.ir.nir, nir_lower_point_size_mov,
                          point_size_state);
               finalize = true;
            }
```
As seen above, there's a call to `_mesa_add_state_reference()` with the pointsize data from where I copied it from the vertex shader callsite, so this is definitely being added to the shader parameters.

## Are These Parameters Being Uploaded?
A quick breakpoint on `st_update_gs_constants()`, which handles uploading the constant buffer data for geometry shaders, revealed that no, there was no upload taking place.

A quick hop over to [st_atom_list.h](https://gitlab.freedesktop.org/mesa/mesa/-/blob/master/src/mesa/state_tracker/st_atom_list.h) revealed that this function is called when `ST_NEW_GS_CONSTANTS` is set in `struct st_program::affected_states`, but this would leave out the case of the codepath being triggered by a tessellation evaluation shader, so really it should be `ST_NEW_CONSTANTS` to be more generic.

On my way out, I decided to check on the original vertex shader handling of the pass, and the state was similarly not being explicitly updated, instead relying on other parts of the code to have initiated an upload of the constant buffers.

In summary, when you copy code that has bugs which happen to not exhibit themselves in the original location, you're basically adding more bugs.
