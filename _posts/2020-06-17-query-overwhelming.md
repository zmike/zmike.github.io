---
published: false
---
## The overflow problem

In past posts I touched on the issue of query pool overflows. The gist of the issue, again, is that each "query" object exposed higher up to the OpenGL API is actually a query pool in Vulkan. Every time an OpenGL query is started or resumed, this consumes a query from the pool. As such, any application which attempts to toggle the active state of a query too many times will end up losing data when the pool gets reset.

# Solution
It wasn't the most difficult of problems. I dove in with the idea of storing partial query results onto the `struct zink_query` object. At `zink_create_query`, the `union pipe_query_result` object gets zeroed, and it can then have partial results concatenated to it. In code, it looks more or less like:

```c
static bool
get_query_result(struct pipe_context *pctx,
                      struct pipe_query *q,
                      bool wait,
                      union pipe_query_result *result)
{
   struct zink_screen *screen = zink_screen(pctx->screen);
   struct zink_query *query = (struct zink_query *)q;
   VkQueryResultFlagBits flags = 0;

   if (wait)
      flags |= VK_QUERY_RESULT_WAIT_BIT;

   if (query->use_64bit)
      flags |= VK_QUERY_RESULT_64_BIT;

   // the below lines added for overflow handling
   if (result != &query->big_result) {
      memcpy(result, &query->big_result, sizeof(query->big_result));
      util_query_clear_result(&query->big_result, query->type);
   } else
      flags |= VK_QUERY_RESULT_PARTIAL_BIT;
```
So when calling this from the application, the result pointer isn't the inlined query result, and the existing result data gets stored to it, then the latest query data is concatenated onto it once it's fetched from Vulkan.

When calling internally from `end_query` just before resetting the pool, the partial bit is set, which permits the return of "whatever results are available". Those get stored onto the inline results struct that gets passed for this case, and this process repeats as many times as necessary until the query is destroyed or the user fetches the results.