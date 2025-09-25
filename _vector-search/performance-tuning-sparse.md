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

A good thing of SEISMIC is that it supports real-time trade-off controlling when users conduct a query. This means that users do not have to re-index if they want to change the balance between search accuracy and query performance. SEISMIC achieves this by six key parameters that affect different aspects of the algorithm:

### Index parameters

These parameters affect index construction and memory usage:

- **`n_postings`**: Maximum documents per posting list (λ parameter)

If a small `n_postings` is set, more aggresive pruning will be applied to the inverted index, which means that fewer document identifiers are kept in one posting list. Reducing this parameter will accelerate index building time and query time but reduce query recall. If you do not specify this parameter, SEISMIC algorithm will decide this value based on $$0.0005 \times \text{document count}$$

- **`cluster_ratio`**: Ratio to determine cluster count

After pruning, there will be `cluster_ratio` $$*$$ `document_count` in each posting list. A higher `cluster_ratio` will lead to more clusters, higher query recall, longer index builing time, and larger query latency.

- **`summary_prune_ratio`**: Ratio for pruning summary vectors (α parameter)

This parameter controls how many tokens will be kept in `summary` of each cluster, where `summary` is used to determine whether a cluster should be examined during query. If you are using different embedding models whose number of tokens greatly vary, you can consider change this parameter. Higher `summary_prune_ratio` will keep more tokens inside `summary`.

- **`approximate_threshold`**: Document threshold for SEISMIC activation

This parameter will control whether to activate SEISMIC algorithm within each segment. When you have more documents in total, the number of documents in one segment will tends to increase. At this time, you might want to increase this threshold to prevent repeating cluster building when small segments merge together. This parameter matters especially when you do not `force_merge` all segments into one, as those small segments will fall back to rank features mode.

### Query parameters

These parameters affect search performance and recall:

- **`top_n`**: Query token pruning limit

In SEISMIC algorithm, a query's all tokens will be pruned to only keep `top_n` ones based on their weights. This parameter will dramastically affect the balance between search performance (latency) and query accuracy (recall). Higher `top_n` will bring with higher accuracy and latency.

- **`heap_factor`**: Recall vs performance tuning multiplier

Every time when SEISMIC determines whether to examine a cluster, it compares potential cluster's score with current queue top's score dividing by `heap_factor`. Larger `heap_factor` will push SEISMIC algorithm to examine more clusters, resulting in higher accuracy but slower query speed. This parameter is more fine-grained compared with `top_n`, which help you to slightly tune the trade-off between accuracy and latency.

## Performance tuning strategies

### Optimizing for high recall
Taking 8.8M MS MARCO dataset as an example, recommended values are shown in parentheses.

- Higher `n_postings` (6000): Allows more documents per cluster, reducing information loss
- Higher `cluster_ratio` (0.15): Creates more clusters, providing finer granularity
- Higher `summary_prune_ratio` (0.5): Retains more information in summary vectors
- Higher `top_n` (5): Considers more query tokens during search
- Higher `heap_factor` (1.2): Explores more candidates during search

### Optimizing for low latency
Taking 8.8M MS MARCO dataset as an example, recommended values are shown in parentheses.

- Lower `n_postings` (3000): Smaller posting lists mean faster traversal
- Lower `cluster_ratio` (0.08): Fewer clusters reduce computational overhead
- Lower `summary_prune_ratio` (0.3): More aggressive pruning reduces memory usage
- Lower `top_n` (3): Processes fewer query tokens
- Lower `heap_factor` (0.9): Explores fewer candidates, reducing computation





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

## Performance benchmarking

### Establish baseline metrics

Before tuning, measure your baseline performance:

1. **Recall metrics**: Use a ground truth dataset to measure recall@k
2. **Latency metrics**: Measure p50, p95, and p99 query latencies
3. **Memory usage**: Monitor JVM heap and off-heap memory usage
4. **Indexing performance**: Measure indexing throughput and time

### Iterative tuning process

Follow this systematic approach:

1. **Start with balanced configuration**
2. **Identify primary bottleneck** (recall vs latency vs memory)
3. **Adjust one parameter at a time**
4. **Measure impact on all metrics**
5. **Iterate until acceptable trade-offs are achieved**

### A/B testing recommendations

When testing parameter changes:

- Use representative query workloads
- Test with production-like data volumes
- Measure performance over extended periods
- Consider query pattern variations (peak vs off-peak)

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
4. Increase `approximate_threshold` to delay activation


## Next steps

- [Sparse ANN configuration reference]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration/)
- [Vector search performance monitoring]({{site.url}}{{site.baseurl}}/monitoring-your-cluster/pa/)