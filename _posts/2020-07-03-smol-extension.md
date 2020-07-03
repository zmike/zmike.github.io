---
published: false
---
## ARB_arrays_of_arrays

I recently came across a number of failing tests where the problem was related to variable sizing in a shader. Check out this beaut:

```glsl
#version 130
#extension GL_ARB_separate_shader_objects: require
#extension GL_ARB_arrays_of_arrays: require

out vec4 out_color;

uniform sampler2D s2[2][2];
uniform sampler3D s3[2][2];

void main()
{
    out_color = texture(s2[1][1], vec2(0)) + texture(s3[1][1], vec3(0));
}
```
When I checked out the corresponding spec, it seems that there's no limitation to array nesting like this, and there's not any handling for arrays of arrays in Zink at present.

Thus I entered the magic of `struct glsl_type` and its many, many, many helper/wrapper functions. Zink has many checks for `glsl_type_is_array(var->type)` when processing variables to find arrays, but there's no checks for arrays of arrays, which was a problem.

Thankfully, as is usually the case in mesa development, someone has had this problem before, and so there's `glsl_get_aoa_size()` for getting the flattened size of an array in these cases. By using the return from this instead of `glsl_get_length()` in these places, Zink can now support this extension.