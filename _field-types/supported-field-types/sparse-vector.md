---
layout: default
title: Sparse-Vector field type
nav_order: 60
has_children: false
parent: Supported field types
---

# Sparse-Vector field type
**Introduced 3.3**
{: .label .label-purple }

Users can index and search with a sparse index. The sparse-vector field boosts the search efficiency for sparse vectors via approximate search algorithms.

For more information, see [Sparse ANN]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic.md).
    
## Sparse-Vector

The sparse vector is represented like a map, where the key is a token and the value is a positive [float]({{site.url}}{{site.baseurl}}/opensearch/supported-field-types/numeric/) value indicating the token weight. 

### Example

Create a mapping with a sparse vector field.

```json
PUT sparse-vector-index
{
  "settings": {
    "index": {
      "sparse": true
    },
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
}
```
{% include copy-curl.html %}

To use sparse-vector field, you need to specify the index setting `index.sparse` to be `true`
{: .note }

For hyper-parameter configuration, you can refer to [`Sparse ANN configuration`]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration.md)  
{: .note }

Index three documents with a sparse_vector field:

```json
PUT sparse-vector-index/_doc/1
{
  "sparse_embedding" : {
    "1000": 0.1
  }
}
```
{% include copy-curl.html %}

```json
PUT sparse-vector-index/_doc/2
{
  "sparse_embedding" : {
    "2000": 0.2
  }
}
```
{% include copy-curl.html %}

```json
PUT sparse-vector-index/_doc/3
{
  "sparse_embedding" : {
    "3000": 0.3
  }
}
```
{% include copy-curl.html %}

## Neural Sparse query

Using a neural-sparse query, you can query the sparse index either by raw vectors or query_text

```json
GET /sparse-vector-index/_search
{
  "query": {
    "neural_sparse": {
      "sparse_embedding": {
        "query_tokens": {
          "1055": 20
        },
        "method_parameters": {
          "heap_factor": 1.0,
          "top_n": 10,
          "k": 10
        }
      }
    }
  }
}
```
{% include copy-curl.html %}

```json
GET /seismic-documents/_search
{
  "query": {
    "neural_sparse": {
      "sparse_embedding": {
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

For query parameter configuration, you can refer to [`Sparse ANN configuration`]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration.md) and [`Sparse performance tuning`]({{site.url}}{{site.baseurl}}/vector-search/performance-tuning-sparse.md). 
{: .note }

