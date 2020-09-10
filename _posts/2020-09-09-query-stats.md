---
published: true
---
## Long, Long Day

I fell into the abyss of query code again, so this is just a short post to (briefly) touch on the adventure that is pipeline statistics queries.

Pipeline statistics queries include a [collection of statistics](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkQueryPipelineStatisticFlagBits.html) about things that happened while the query was active, such as the count of shader invocations.

Thankfully, these weren't too difficult to plug into the ever-evolving zink query architecture, as my previous adventure into QBOs (more on this another time) ended up breaking out a lot of helper functions for various things that simplified the process.

## Mapping
The first step, as always, is figuring out how to map gallium API to vulkan. The below table handles the basic conversion for single query types:
```c
unsigned map[] = {
   [PIPE_STAT_QUERY_IA_VERTICES] = VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_VERTICES_BIT,
   [PIPE_STAT_QUERY_IA_PRIMITIVES] = VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_PRIMITIVES_BIT,
   [PIPE_STAT_QUERY_VS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_VERTEX_SHADER_INVOCATIONS_BIT,
   [PIPE_STAT_QUERY_GS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_GEOMETRY_SHADER_INVOCATIONS_BIT,
   [PIPE_STAT_QUERY_GS_PRIMITIVES] = VK_QUERY_PIPELINE_STATISTIC_GEOMETRY_SHADER_PRIMITIVES_BIT,
   [PIPE_STAT_QUERY_C_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_CLIPPING_INVOCATIONS_BIT,
   [PIPE_STAT_QUERY_C_PRIMITIVES] = VK_QUERY_PIPELINE_STATISTIC_CLIPPING_PRIMITIVES_BIT,
   [PIPE_STAT_QUERY_PS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_FRAGMENT_SHADER_INVOCATIONS_BIT,
   [PIPE_STAT_QUERY_HS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_TESSELLATION_CONTROL_SHADER_PATCHES_BIT,
   [PIPE_STAT_QUERY_DS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_TESSELLATION_EVALUATION_SHADER_INVOCATIONS_BIT,
   [PIPE_STAT_QUERY_CS_INVOCATIONS] = VK_QUERY_PIPELINE_STATISTIC_COMPUTE_SHADER_INVOCATIONS_BIT
};
```
Pretty straightforward.

With this in place, it's worth mentioning that OpenGL has facilities for performing either "all" statistics queries, which include all the above types, or "single" statistics queries, which is just one of the types.

Building on that, I'm going to reach [way back]({{site.url}}/querying-xfb) to the original loop that I've been using for handling query results. That's now its own helper function called `check_query_results()`. It's invoked from `get_query_result()` like so:
```c
int result_size = 1;
   /* these query types emit 2 values */
if (query->vkqtype == VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT ||
    query->type == PIPE_QUERY_PRIMITIVES_GENERATED ||
    query->type == PIPE_QUERY_PRIMITIVES_EMITTED)
   result_size = 2;
else if (query->type == PIPE_QUERY_PIPELINE_STATISTICS)
   result_size = 11;

if (query->type == PIPE_QUERY_PIPELINE_STATISTICS)
   num_results = 1;
for (unsigned last_start = query->last_start; last_start + num_results <= query->curr_query; last_start++) {
   /* verify that we have the expected number of results pending */
   assert(num_results <= ARRAY_SIZE(results) / result_size);
   VkResult status = vkGetQueryPoolResults(screen->dev, query->query_pool,
                                           last_start, num_results,
                                           sizeof(results),
                                           results,
                                           sizeof(uint64_t),
                                           flags);
   if (status != VK_SUCCESS)
      return false;

   if (query->type == PIPE_QUERY_PRIMITIVES_GENERATED) {
      status = vkGetQueryPoolResults(screen->dev, query->xfb_query_pool[0],
                                              last_start, num_results,
                                              sizeof(xfb_results),
                                              xfb_results,
                                              2 * sizeof(uint64_t),
                                              flags | VK_QUERY_RESULT_64_BIT);
      if (status != VK_SUCCESS)
         return false;

   }

   check_query_results(query, result, num_results, result_size, results, xfb_results);
}
```
Beautiful, isn't it?

For the case of any xfb-related query, 2 result values are returned (and then also `PRIMITIVES_GENERATED` gets its own separate xfb pool for correctness), while most others only get 1 result value.

And then there's `PIPELINE_STATISTICS`, which gets *eleven*. Since this is a weird result size to handle, I decided to just grab the results for each query one at a time in order to avoid any kind of craziness with allocating enough memory for the actual result values. Plenty of time for that later.

The handling for the results is fun too:
```c
case PIPE_QUERY_PIPELINE_STATISTICS: {
   uint64_t *statistics = (uint64_t*)&result->pipeline_statistics;
   for (unsigned j = 0; j < 11; j++)
      statistics[j] += results[i + j];
   break;
}
```
Each bit set in the query object returns its own query result, so there's 11 of them that need to be accumulated for each query result.

Surprisingly though, that's the craziest part of implementing this. Not really much compared to some of the other insanity.

## Corrections
The other day, I made wild claims about zink being the fastest graphics driver in history.

I'm not going to say they were inaccurate.

What I am going to say is that I had a great chat with well-known, long-time mesa developer Kenneth Graunke, who pointed out that most of the time differences in the softfp64 tests were due to NIR optimizations related to loop unrolling and the different settings used between IRIS and zink there.

He also helpfully pointed out that I forgot to set a number of important pipe caps, including the one that holds the value for `gl_MaxVaryingComponents`. As a result, zink has been advertising a [very illegal](https://www.youtube.com/watch?v=I7Tps0M-l64&hd=1) and too-small number of varyings, which made the shaders smaller, which made us faster.

This has since been fixed, and zink is now fully compliant with the required GL specs for this value.

However.

This has only increased the time of the one test I showcased from ~2 seconds to ~12 seconds.

Due to different mechanics between GLSL and SPIRV shader construction, all XFB outputs have to be explicitly specified when they're converted in zink, and they count against the regular output limits. As a result, it's necessary to reserve half of the driver's advertised vertex outputs for potential XFB usage. This leaves us with much fewer available output slots for user shaders in order to preserve XFB functionality.
