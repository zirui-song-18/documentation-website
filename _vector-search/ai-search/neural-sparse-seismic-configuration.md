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

SEISMIC is an **A**pproximate **N**earest **N**eighbor (ANN) algorithm designed to accelerate neural sparse vector queries by organizing documents into clusters with summary vectors, enabling efficient pruning during search operations. Unlike traditional neural sparse search that relies solely on inverted indexes, SEISMIC maintains both a forward index of sparse vectors and clustered posting lists to achieve significant query performance improvements.

## Prerequisites

Before configuring SEISMIC, ensure you have:

- OpenSearch 2.11 or later with the neural-search plugin installed
- Sufficient cluster resources (CPU cores & JVM memory) for SEISMIC index building and caching

## Index configuration

### Enable SEISMIC for an index

To use SEISMIC, you must enable sparse setting at the index level:

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

### Hybrid indexing behavior

SEISMIC uses a hybrid approach based on segment size:

- **Small segments** (below `approximate_threshold`): Use traditional rank features with existing neural sparse query logic
- **Large segments** (at or above `approximate_threshold`): Apply SEISMIC clustering algorithm with new query logic

This ensures optimal performance across different data scales while maintaining indexing efficiency for smaller datasets.

## Field mapping configuration

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

## Query configuration

### Basic SEISMIC query

Query SEISMIC fields using the enhanced `neural_sparse` query:

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
### Raw Vector SEISMIC query
In addition, you can also prepare sparse vectors in advance so that you can send a raw vector as a query. Please note that you should use int token id here instead of raw text.
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
| `n_postings` | Integer | No | Maximum documents per posting list (λ parameter) | `0.0005* doc count`* | $$(0, \infty)$$ | `4000` |
| `cluster_ratio` | Float | No | Ratio to determine cluster count | `0.1` | $$(0, 1)$$ | `0.15` |
| `summary_prune_ratio` | Float | No | Ratio for pruning summary vectors (α parameter) | `0.4` | $$(0, 1]$$ | `0.3` |
| `approximate_threshold` | Integer | No | Document threshold for SEISMIC activation | `1000000` | $$[0, \infty)$$ | `500000` |

*doc count here is segment level\

### Query parameters

| Parameter | Type | Required | Description | Default | Range | Example |
|-----------|------|----------|-------------|---------|---------|---------|
| `k` | Integer | Yes | Number of results to return | - | $$>0$$ | `10` |
| `model_id` | String | Yes** | Sparse encoding model ID | - | - | `"abc123def456"` |
| `query_text` | String | No* | Text to encode and search | - | - | `"machine learning"` |
| `query_tokens` | Object | No* | Pre-encoded token map | - | - | `{"token": 0.8}` |
| `top_n` | Integer | No | Query token pruning limit | `10` | $$>0$$ | `15` |
| `heap_factor` | Float | No | Recall vs performance tuning | `1.0` | $$>0$$ | `1.5` |
| `filter` | Object | No | Pre-filtering query | - | - | `{"term": {...}}` |

*Either `query_text` or `query_tokens` is required\
**Required when using `query_text`

## Cluster settings

### Thread pool configuration

If your hardware has multiple available cores, you can allow multiple threads when building the clustered inverted index. When you have more than one thread working, the clustering work of every posting list will be distributed to available threads, which accelerate the clustering process. Determine the number of threads by tuning the `plugins.neural_search.sparse.algo_param.index_thread_qty` setting

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.neural_search.sparse.algo_param.index_thread_qty": 4
  }
}
```

### Memory and caching settings
Here is an example to call the circuit breaker cluster setting API:
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
As a kind of ANN algorithm, SEISMIC provides users with an opportunity to balance the trade-off between how accurate search results are and how fast search process can be. In short, you can tune balance between recall and latency with following parameter settings.
### Optimizing for recall vs speed

- **Higher recall**: Increase `heap_factor`, increase `cluster_ratio` 
- **Higher speed**: Decrease `heap_factor`, decrease `n_postings`
- **Balanced**: Use default values and adjust based on benchmarking

### Memory optimization

- Reduce `n_postings` for lower memory usage
- Increase `approximate_threshold` to delay SEISMIC activation
- Configure appropriate cache settings based on available memory

### Indexing performance

- Adjust `plugins.neural_search.sparse.algo_param.index_thread_qty` based on CPU cores
- Use fewer shards for better clustering efficiency
- Consider [batch]({{site.url}}{{site.baseurl}}/ml-commons-plugin/remote-models/batch-ingestion/) size in ingest pipelines

## Limitations and considerations

### Current limitations

- **Codec restriction**: Indices with `index.sparse: true` cannot use k-NN dense vector fields
- **Not support two-phase**: Currently SEISMIC does not support hybrid search with two-phase pipeline
- **Memory requirements**: SEISMIC requires additional memory for forward index and cluster caching
- **Indexing overhead**: Clustering process adds computational cost during indexing

### Best practices

- Start with default parameters and tune based on your specific dataset
- Monitor memory usage and adjust cache settings accordingly
- Use SEISMIC for large-scale datasets where query performance is critical
- Consider the trade-off between indexing time and query performance

## Troubleshooting

### Common issues

**SEISMIC not activating**
- Check that `index.sparse: true` is set
- Verify segment size exceeds `approximate_threshold`
- Confirm field type is `sparse_vector`

**High memory usage**
- Reduce `n_postings` parameter
- Adjust cache settings
- Consider increasing `approximate_threshold`

**Slow SEISMIC query performance**
- Tune `heap_factor` and `top_n` parameters
- Check `approximate_threshold` and [`_cat/segments`]({{site.url}}{{site.baseurl}}/api-reference/cat/index/) API.

**Indexing failures**
- Monitor thread pool settings
- Check available memory during indexing
- Verify sparse vector generation in pipeline


## Next steps

- [Sparse ANN performance tuning]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse/)
- [Hybrid search with SEISMIC]({{site.url}}{{site.baseurl}}/search-plugins/hybrid-search/)