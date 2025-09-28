---
layout: default
title: Sparse ANN configuration
parent: Sparse ANN
grand_parent: Neural sparse search
great_grand_parent: AI search
nav_order: 10
has_math: true
---

# Sparse ANN configuration

This page provides comprehensive configuration guidance for Sparse ANN, especially the SEISMIC algorithm, in OpenSearch neural sparse search.

## Prerequisites

Before configuring Sparse ANN, ensure you have:

- OpenSearch 3.3 or later with the neural-search plugin installed
- Sufficient cluster resources (CPU cores & JVM memory) for Sparse ANN index building and caching. See [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/).

## Step 1: Create an index

To use Sparse ANN, you must enable sparse setting at the index level by setting `"sparse": true`

In addition, there are some special parameters in a mapping field. You can specify what settings you want to use, such as `n_postings`, `cluster_ratio`, `summary_prune_ratio`, and `approximate_threshold`. More details can be seen in More details can be seen in [Sparse ANN index setting]({{site.url}}{{site.baseurl}}/field-types/supported-field-types/index/)

### Example
```json
PUT /sparse-ann-documents
{
  "settings": {
    "index": {
      "sparse": true,
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  },
  "mappings": {
    "properties": {
      "sparse_embedding": {
        "type": "sparse_vector",
        "method": {
          "name": "seismic",
          "parameters": {
            "n_postings": 4000,
            "cluster_ratio": 0.1,
            "summary_prune_ratio": 0.4,
            "approximate_threshold": 1000000
          }
        }
      }
    }
  }
}
```
{% include copy-curl.html %}

## Step 2: Ingest data

### Step 2.a: Directly ingest data

After a Sparse ANN index is successfully created, you can ingest data into it

```json
POST _bulk
{ "create": { "_index": "sparse-ann-documents", "_id": "0" } }
{ "sparse_embedding": {"10": 0.85, "23": 1.92, "24": 0.67, "78": 2.54, "156": 0.73} }
{ "create": { "_index": "sparse-ann-documents", "_id": "1" } }
{ "sparse_embedding": {"3": 1.22, "19": 0.11, "21": 0.35, "300": 1.74, "985": 0.96} }
```
{% include copy-curl.html %}

### Step 2.b: Force merge (Optional)

To obtain better query performance, you can choose to merge all segments into one. More details can be seen in [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)

```json
POST /sparse-ann-documents/_forcemerge?max_num_segments=1
```
{% include copy-curl.html %}

## Step 3: Conduct a query

Now, you can prepare a query to revtrieve information from the index you just built. Please note that you should not combine Sparse ANN with [two-phase]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/neural-sparse-query-two-phase-processor/) pipeline.

### Natural Language query

Query Sparse ANN fields using the enhanced `neural_sparse` query:

```json
GET /sparse-ann-documents/_search
{
  "query": {
    "neural_sparse": {
      "content_vector": {
        "query_text": "machine learning algorithms",
        "model_id": "your_sparse_model_id",
        "k": 10,
        "top_n": 10,
        "heap_factor": 1.0
      }
    }
  }
}
```
{% include copy-curl.html %}

### Raw vector Sparse ANN query
In addition, you can also prepare sparse vectors in advance so that you can send a raw vector as a query. Please note that you should use tokens in a form of Integer here instead of raw text.
```json
GET /sparse-ann-documents/_search
{
  "query": {
    "neural_sparse": {
      "sparse_embedding": {
        "query_tokens": {
          "1055": 1.7,
          "2931": 2.3
        },
        "method_parameters": {
          "heap_factor": 1.2,
          "top_n": 6,
          "k": 10
        }
      }
    }
  }
}
```
{% include copy-curl.html %}

## Cluster settings

### Thread pool configuration

When building clustered inverted index structure, it requires intensive computations. Our algorithm uses a threadpool to building clusters in parallel. The default value of the threadpool is 1. You can adjust this `plugins.neural_search.sparse.algo_param.index_thread_qty` setting to tune the threadpool size to use more CPU cores to reduce index building time

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.neural_search.sparse.algo_param.index_thread_qty": 4
  }
}
```
{% include copy-curl.html %}

### Memory and caching settings
Sparse ANN is equipped a circuit breaker to prevent consuming too much memory space, so users do not need to worry about affecting other OpenSearch services. The default value of `circuit_breaker.limit` is $$10\%$$, and you can set a different limit value to control its memory size and cache performance. Here is an example to call the circuit breaker cluster setting API:
```json
PUT _cluster/settings
{
  "persistent": {
    "plugins.neural_search.circuit_breaker.limit": "30%"
  }
}
```
{% include copy-curl.html %}

More details can be seen in [Neural Search API]({{site.url}}{{site.baseurl}}/vector-search/api/neural/).

## Performance tuning
Sparse ANN provides users with an opportunity to balance the trade-off between how accurate search results are and how fast search process can be. In short, you can tune balance between recall and latency with following parameter settings. Check guidance in [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)

## Common issues

**Slow query performance**
1. Sparse ANN is not activated:
    - Check that `index.sparse: true` is set
    - Verify segment size exceeds `approximate_threshold`
    - Confirm field type is `sparse_vector`
2. Improper parameters:
    - Decrease `heap_factor` and `top_n` parameters
    - Change `approximate_threshold` on a new index if needed

**High memory usage**
- Reduce `n_postings` parameter
- Adjust cache settings
- Consider increasing `approximate_threshold`

**Indexing failures**
- Monitor thread pool settings
- Check available memory during indexing
- Verify sparse vector generation in pipeline


## Next steps

- [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)