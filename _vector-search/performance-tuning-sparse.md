---
layout: default
title: Sparse performance tuning
nav_order: 30
parent: Performance tuning
has_math: true
---

# Sparse performance tuning

This page provides comprehensive performance tuning guidance for the Sparse ANN [SEISMIC]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic/) algorithm in OpenSearch neural sparse search. SEISMIC offers multiple parameters that allow you to balance the trade-off between search recall (accuracy) and query performance (latency).

## Understanding SEISMIC parameters

A good thing of SEISMIC is that it supports real-time trade-off controlling when users conduct a query by those search-time parameters. This means that users do not have to re-index if they want to change the balance between search accuracy and query performance. In total, SEISMIC employs six key parameters that affect different aspects of the algorithm:

### Index parameters

These parameters affect index construction and memory usage:

- **`n_postings`**: Maximum documents per posting list (λ parameter)

If a small `n_postings` is set, more aggresive pruning will be applied to the inverted index, which means that fewer document identifiers are kept in one posting list. Reducing this parameter will accelerate index building time and query time but reduce query recall. If you do not specify this parameter, SEISMIC algorithm will decide this value based on $$0.0005 \times \text{document count}$$

- **`cluster_ratio`**: Ratio to determine cluster count

After pruning, there will be `cluster_ratio` $$*$$ `document_count` in each posting list. A higher `cluster_ratio` will lead to more clusters, higher query recall, longer index builing time, and larger query latency.

- **`summary_prune_ratio`**: Ratio for pruning summary vectors (α parameter)

This parameter controls how many tokens will be kept in `summary` of each cluster, where `summary` is used to determine whether a cluster should be examined during query. If you are using different embedding models whose number of tokens greatly vary, you can consider change this parameter. Higher `summary_prune_ratio` will keep more tokens inside `summary`.

- **`approximate_threshold`**: Document threshold for SEISMIC activation

This parameter will control whether to activate SEISMIC algorithm within each segment. When you have more documents in total, the number of documents in one segment will tend to increase. At this time, you might want to increase this threshold to prevent repeating cluster building when small segments merge together. This parameter matters especially when you do not `force_merge` all segments into one, as those small segments will fall back to rank features mode.

### Query parameters

These parameters affect search performance and recall:

- **`top_n`**: Query token pruning limit

In SEISMIC algorithm, a query's all tokens will be pruned to only keep `top_n` ones based on their weights. This parameter will dramastically affect the balance between search performance (latency) and query accuracy (recall). Higher `top_n` will bring with higher accuracy and latency.

- **`heap_factor`**: Recall vs performance tuning multiplier

Every time when SEISMIC determines whether to examine a cluster, it compares potential cluster's score with current queue top's score dividing by `heap_factor`. Larger `heap_factor` will push SEISMIC algorithm to examine more clusters, resulting in higher accuracy but slower query speed. This parameter is more fine-grained compared with `top_n`, which help you to slightly tune the trade-off between accuracy and latency.

## Optimize beyond parameters

### Building clusters

If your platform supports multiple processors, you can adjust the number of threads (`index_thread_qty`) during building clusters. The default value of `index_thread_qty` is 1, and you can change this setting according to [Neural-Search cluster settings]({{site.url}}{{site.baseurl}}/vector-search/ai-search/needs-to-be-implemented/). Higher `index_thread_qty` will reduce `force_merge` time when SEISMIC is activated, while comsuming more resources at the same time.

### Query when no data in cache

If you plan to conduct a query when there is no data in cache (e.g. reboot OpenSearch Cluster), search latency will be quite high as more time will be spent on I/O to load data. To address this issue, you are welcome to call `warmup` API according to [Neural-Search cluster settings]({{site.url}}{{site.baseurl}}/vector-search/ai-search/needs-to-be-implemented/). This API will automatically load data from disk to cache, making sure the following query can have best performance. In contrast, you can also call `clear_cache` API to free memory usage.

### Monitor memory usage

Use circuit breaker settings to monitor and control memory:

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.neural_search.circuit_breaker.limit": "30%",
    "plugins.neural_search.circuit_breaker.overhead": "1.01"
  }
}
```
More details can be seen in [Neural-Search cluster settings]({{site.url}}{{site.baseurl}}/vector-search/ai-search/needs-to-be-implemented/).


## Troubleshooting performance issues

### Low recall symptoms and solutions

**Symptoms:**
- Relevant documents not appearing in top results
- Inconsistent search quality across queries

**Solutions:**
1. Increase `heap_factor`
2. Increase `top_n`
3. Increase `summary_prune_ratio`
4. Increase `n_postings`

### High latency symptoms and solutions

**Symptoms:**
- Query response times > 100ms consistently
- Timeout errors during peak traffic

**Solutions:**
1. Decrease `heap_factor`
2. Decrease `top_n`
3. Decrease `n_postings`
4. Decrease `summary_prune_ratio`

### Memory pressure symptoms and solutions

**Symptoms:**
- Circuit breaker activation
- OutOfMemoryError exceptions
- Slow garbage collection

**Solutions:**
1. Decrease `n_postings` significantly
2. Decrease `summary_prune_ratio`
3. Decrease `cluster_ratio`
4. Increase `approximate_threshold` to delay SEISMIC activation


## Next steps

- [Sparse ANN configuration reference]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration/)
- [Vector search performance monitoring]({{site.url}}{{site.baseurl}}/monitoring-your-cluster/pa/)