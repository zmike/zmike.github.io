---
published: true
---
## Queries Again

I've talked in the past about XFB, and I've talked about queries, but I've never spent much time talking about XFB queries.

That's going to change.

XFB is not great, and queries aren't something I'm too fond of at this point after rewriting the handling so many times, but it was only the other day, while handling streams for XFB3/ARB_gpu_shader5 (humblebrag) that I realized I had been in the infant area of the playground until this moment.

## Enter GL_PRIMITIVES_GENERATED
Yes, it's this query. This one query which behaves differently based on whether XFB is currently active. And in Vulkan terms, this difference is nightmarish for anyone who wants to maintain a "simple" query implementation.

Here's the gist:
* When XFB is active, GL_PRIMITIVES_GENERATED needs to return the XFB extension query data for primitives generated
* When XFB is not active, GL_PRIMITIVES_GENERATED needs to return pipeline statistics query data

But hold on. It gets worse.

For the second case, there's two subcases:
* If a geometry shader is present, `VK_QUERY_PIPELINE_STATISTIC_GEOMETRY_SHADER_PRIMITIVES_BIT` is the value that needs to be returned
* Otherwise, it's `VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_PRIMITIVES_BIT`

## Ughhhhhhhhhh
I don't really have much positive to say here other than "it works".

The current solution is to maintain two separate `VkQueryPool` objects for these types of queries, keep them both in sync as far as active/not-active is concerned, and then use results based on `have_xfb ? use_xfb : (have_gs ? use_gs : use_vs)`, and it's about as gross as you'd imagine.
