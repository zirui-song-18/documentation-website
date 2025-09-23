---
layout: default
title: SEISMIC configuration
parent: SEISMIC algorithm
grand_parent: Neural sparse search
great_grand_parent: AI search
nav_order: 10
---

# SEISMIC configuration

This page provides comprehensive configuration guidance for the SEISMIC algorithm in OpenSearch neural sparse search.

SEISMIC is an approximate nearest neighbor (ANN) algorithm designed to accelerate neural sparse vector queries by organizing documents into clusters with summary vectors, enabling efficient pruning during search operations. Unlike traditional neural sparse search that relies solely on inverted indexes, SEISMIC maintains both a forward index of sparse vectors and clustered posting lists to achieve significant query performance improvements.

## Prerequisites

Before configuring SEISMIC, ensure you have:

- OpenSearch 2.15 or later with the neural-search plugin installed
- Sufficient cluster resources for SEISMIC index building and caching
- Understanding that SEISMIC is an indexing and query optimization technique, not a model

## Index configuration

### Enable SEISMIC for an index

To use SEISMIC, you must enable sparse ANN at the index level:

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

Configure sparse vector fields with the `sparse_tokens` type and SEISMIC algorithm:

```json
{
  "mappings": {
    "properties": {
      "sparse_embedding": {
        "type": "sparse_tokens",
        "method": {
          "name": "seismic",
          "parameters": {
            "n_postings": 50,
            "cluster_ratio": 0.1,
            "summary_prune_ratio": 0.4,
            "approximate_threshold": 100000
          }
        }
      }
    }
  }
}
```

### Multiple sparse fields

You can configure multiple sparse ANN fields in the same index:

```json
{
  "mappings": {
    "properties": {
      "title_vector": {
        "type": "sparse_tokens",
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
        "type": "sparse_tokens",
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

## Parameters reference

### Index settings

| Parameter | Type | Required | Description | Default | Example |
|-----------|------|----------|-------------|---------|---------|
| `index.sparse` | Boolean | Yes | Enable sparse ANN for the index | `false` | `true` |

### Algorithm parameters

| Parameter | Type | Required | Description | Default | Example |
|-----------|------|----------|-------------|---------|---------|
| `name` | String | Yes | Algorithm name | - | `"seismic"` |
| `n_postings` | Integer | No | Maximum documents per posting list (λ parameter) | `0.0005* doc count` | `4000` |
| `cluster_ratio` | Float | No | Ratio to determine cluster count | `0.1` | `0.15` |
| `summary_prune_ratio` | Float | No | Ratio for pruning summary vectors (α parameter) | `0.4` | `0.3` |
| `approximate_threshold` | Integer | No | Document threshold for SEISMIC activation | `1000000` | `500000` |

### Query parameters

| Parameter | Type | Required | Description | Default | Example |
|-----------|------|----------|-------------|---------|---------|
| `query_text` | String | No* | Text to encode and search | - | `"machine learning"` |
| `query_tokens` | Object | No* | Pre-encoded token map | - | `{"token": 0.8}` |
| `model_id` | String | Yes** | Sparse encoding model ID | - | `"abc123def456"` |
| `k` | Integer | Yes | Number of results to return | - | `10` |
| `top_n` | Integer | No | Query token pruning limit | `10` | `15` |
| `heap_factor` | Float | No | Recall vs performance tuning | `1.0` | `1.5` |
| `filter` | Object | No | Pre-filtering query | - | `{"term": {...}}` |

*Either `query_text` or `query_tokens` is required
**Required when using `query_text`

## Cluster settings

### Thread pool configuration

If your hardware has multiple available cores, you can allow multiple threads when building the clustered inverted index. Determine the number of threads by tuning the `sparse.algo_param.index_thread_qty` setting

```json
PUT /_cluster/settings
{
  "persistent": {
    "sparse.algo_param.index_thread_qty": 4
  }
}
```

### Memory and caching settings

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.neural_search.seismic.cache.max_size": "2GB",
    "plugins.neural_search.seismic.cache.ttl": "1h"
  }
}
```

## Performance tuning

### Optimizing for recall vs speed

- **Higher recall**: Increase `heap_factor` (1.5-2.0), increase `cluster_ratio` (0.15-0.2)
- **Higher speed**: Decrease `heap_factor` (0.5-0.8), decrease `n_postings` (2000-3000)
- **Balanced**: Use default values and adjust based on benchmarking

### Memory optimization

- Reduce `n_postings` for lower memory usage
- Increase `approximate_threshold` to delay SEISMIC activation
- Configure appropriate cache settings based on available memory

### Indexing performance

- Adjust `sparse.algo_param.index_thread_qty` based on CPU cores
- Use fewer shards for better clustering efficiency
- Consider batch size in ingest pipelines

## Limitations and considerations

### Current limitations

- **Codec restriction**: Indices with `index.sparse: true` cannot use k-NN dense vector fields
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
- Confirm field type is `sparse_ann`

**High memory usage**
- Reduce `n_postings` parameter
- Adjust cache settings
- Consider increasing `approximate_threshold`

**Poor query performance**
- Tune `heap_factor` and `top_n` parameters
- Verify adequate cluster resources

**Indexing failures**
- Monitor thread pool settings
- Check available memory during indexing
- Verify sparse vector generation in pipeline


## Next steps

- [SEISMIC examples and tutorials]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-examples/)
- [Neural sparse search performance benchmarks]({{site.url}}{{site.baseurl}}/benchmarks/neural-sparse/)
- [Hybrid search with SEISMIC]({{site.url}}{{site.baseurl}}/search-plugins/hybrid-search/)