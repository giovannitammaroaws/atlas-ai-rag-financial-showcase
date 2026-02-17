# 02 - Query Cache and Cost Saving

When the same normalized query is repeated with the same parameters, the system serves the response from internal cache instead of triggering a new provider call.

Why it matters:
- lower latency for repeated requests
- lower token usage and lower cost
- consistent answer for identical query inputs

Cache behavior:
1. check recent query signature (query + ticker + model + top-k)
2. if hit: return cached answer immediately
3. if miss: run retrieval + LLM and persist result for future hits

![Query Cache](images/query_cache.png)

[Back to README](../README.md)
