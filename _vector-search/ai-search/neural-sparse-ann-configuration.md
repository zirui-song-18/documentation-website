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

## Step 1: Index configuration

### Enable sparse for an index

To use Sparse ANN, you must enable sparse setting at the index level:

```json
PUT /seismic-documents
{
  "settings": {
    "index": {
      "sparse": true,
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}
```

## Step 2: Field mapping configuration

### SEISMIC field type

Configure sparse vector fields with the `sparse_vector` type and SEISMIC algorithm:

```json
{
  "mappings": {
    "properties": {
      "sparse_embedding": {
        "type": "sparse_vector",
        "method": {
          "name": "seismic",
          "parameters": {
            "n_postings": 300,
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

### Multiple sparse fields

You can configure multiple sparse ANN fields with different hyper-parameters in the same index:

```json
{
  "mappings": {
    "properties": {
      "title_vector": {
        "type": "sparse_vector",
        "method": {
          "name": "seismic",
          "parameters": {
            "n_postings": 2000,
            "cluster_ratio": 0.1,
            "summary_prune_ratio": 0.4,
            "approximate_threshold": 1000000
          }
        }
      },
      "content_vector": {
        "type": "sparse_vector",
        "method": {
          "name": "seismic",
          "parameters": {
            "n_postings": 4000,
            "cluster_ratio": 0.15,
            "summary_prune_ratio": 0.36,
            "approximate_threshold": 2000000
          }
        }
      }
    }
  }
}
```

## Step 3: Query configuration

### Basic Sparse ANN query

Query Sparse ANN fields using the enhanced `neural_sparse` query:

```json
GET /seismic-documents/_search
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
### Raw vector Sparse ANN query
In addition, you can also prepare sparse vectors in advance so that you can send a raw vector as a query. Please note that you should use tokens with integer IDs here instead of raw text.
```json
GET /seismic-documents/_search
{
  "query": {
    "neural_sparse": {
      "sparse_embedding": {
        "query_tokens": {
          "1055": 20
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

## Parameters reference

### Index settings

| Parameter | Type | Required | Description | Default | Example |
|-----------|------|----------|-------------|---------|---------|
| `index.sparse` | Boolean | Yes | Enable sparse ANN for the index | `false` | `true` |

### Algorithm parameters

| Parameter | Type | Required | Description | Default | Range | Example |
|-----------|------|----------|-------------|---------|---------|---------|
| `name` | String | Yes | Algorithm name | - | - | `"seismic"` |
| `n_postings` | Integer | No | Maximum documents per posting list (λ parameter) | `0.0005* doc count`¹ | $$(0, \infty)$$ | `4000` |
| `cluster_ratio` | Float | No | Ratio to determine cluster count | `0.1` | $$(0, 1)$$ | `0.15` |
| `summary_prune_ratio` | Float | No | Ratio for pruning summary vectors (α parameter) | `0.4` | $$(0, 1]$$ | `0.3` |
| `approximate_threshold` | Integer | No | Document threshold for SEISMIC activation | `1000000` | $$[0, \infty)$$ | `500000` |

¹doc count here is segment level

### Query parameters

| Parameter | Type | Required | Description | Default | Range | Example |
|-----------|------|----------|-------------|---------|---------|---------|
| `k` | Integer | Yes | Number of results to return | - | $$>0$$ | `10` |
| `model_id` | String | Yes² | Sparse encoding model ID | - | - | `"abc123def456"` |
| `query_text` | String | No¹ | Text to encode and search | - | - | `"machine learning"` |
| `query_tokens` | Object | No¹ | Pre-encoded token map | - | - | `{"token": 0.8}` |
| `top_n` | Integer | No | Query token pruning limit | `10` | $$>0$$ | `15` |
| `heap_factor` | Float | No | Recall vs performance tuning | `1.0` | $$>0$$ | `1.5` |
| `filter` | Object | No | Pre-filtering query | - | - | `{"term": {...}}` |

¹Either `query_text` or `query_tokens` is required\
²Required when using `query_text`

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

### Memory and caching settings
Sparse ANN may consume much memory space due to its clustered posting list and forward index structure. If you worry about the SEISMIC index consuming too much memory space, you can set a circuit breaker to protect OpenSearch cluster running and other plugin services. Here is an example to call the circuit breaker cluster setting API:
```json
PUT _cluster/settings
{
  "persistent": {
    "plugins.neural_search.circuit_breaker.limit": "30%",
    "plugins.neural_search.circuit_breaker.overhead": "1.01"
  }
}
```
More details can be seen in [Neural Search API]({{site.url}}{{site.baseurl}}/vector-search/api/neural/).

## Performance tuning
Sparse ANN provides users with an opportunity to balance the trade-off between how accurate search results are and how fast search process can be. In short, you can tune balance between recall and latency with following parameter settings. Check guidance in [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)

## Troubleshooting

### Common issues

**Sparse ANN not activating**
- Check that `index.sparse: true` is set
- Verify segment size exceeds `approximate_threshold`
- Confirm field type is `sparse_vector`

**High memory usage**
- Reduce `n_postings` parameter
- Adjust cache settings
- Consider increasing `approximate_threshold`

**Slow Sparse ANN query performance**
- Tune `heap_factor` and `top_n` parameters
- Check `approximate_threshold` and [`_cat/segments`]({{site.url}}{{site.baseurl}}/api-reference/cat/index/) API.

**Indexing failures**
- Monitor thread pool settings
- Check available memory during indexing
- Verify sparse vector generation in pipeline


## Next steps

- [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)
- [Hybrid search with SEISMIC]({{site.url}}{{site.baseurl}}/search-plugins/hybrid-search/)