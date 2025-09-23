---
layout: default
title: SEISMIC algorithm
parent: Neural sparse search
grand_parent: AI search
nav_order: 60
has_children: true
redirect_from:
  - /search-plugins/neural-sparse-seismic/
---

# SEISMIC algorithm
Introduced 3.3.0
{: .label .label-purple }

SEISMIC (**S**pilled Clust**e**ring of **I**nverted Lists with **S**ummaries for **M**aximum **I**nner Produ**c**t Search from "[Efficient Inverted Indexes for Approximate Retrieval over Learned Sparse Representations](https://arxiv.org/abs/2404.18812)" paper) is an **A**pproximate **N**earest **N**eighbor (ANN) algorithm specifically designed to accelerate neural sparse search. Unlike traditional sparse encoding methods, SEISMIC is not an encoding algorithm itself, but rather an indexing and retrieval optimization technique that significantly improves query performance for existing neural sparse vectors. Now, Seismic algorithm is available in the Neural-Search plugin to provide outstanding sparse vector query performance.

SEISMIC addresses the performance challenges that arise when neural sparse search is applied to large-scale datasets with billions of vectors. While neural sparse search traditionally uses inverted indexes for efficiency, query performance can deteriorate substantially at unprecedented data volumes. SEISMIC introduces a novel approach using clustered posting lists and approximate retrieval techniques to maintain fast query speeds even as data scales exponentially.

## How SEISMIC works

SEISMIC operates through a sophisticated two-stage process that optimizes both indexing and querying of neural sparse vectors:

### Indexing stage

During the indexing phase, SEISMIC implements several key optimizations:

1. **Hybrid sparse indexing**: SEISMIC uses a document threshold-based approach where segments with fewer documents than the configured threshold use traditional rank features storage, while larger segments employ the SEISMIC clustering approach.

2. **Posting list clustering**: For each term in the inverted index, SEISMIC:
   - Sorts documents by their token weights in descending order
   - Retains only the top λ (lambda) documents with highest weights
   - Applies a clustering algorithm to group similar documents into β (beta) clusters
   - Generates summary sparse vectors for each cluster, keeping only the top α (alpha) tokens

3. **Forward index maintenance**: SEISMIC maintains both the traditional inverted index and a forward index that stores all sparse vectors for efficient access during query processing.

4. **Clustered posting lists**: Documents are organized into clusters with corresponding cluster summaries, allowing traversal at both cluster and document levels.

### Query processing stage

During query execution, SEISMIC employs an efficient retrieval process:

1. **Cluster-level filtering**: The algorithm first computes dot product scores between the query vector and cluster summary vectors. Only clusters with scores above a dynamic threshold are selected for detailed examination.

2. **Document-level scoring**: For selected clusters, SEISMIC examines individual documents within those clusters, computing exact dot product scores between the query and document vectors retrieved from the forward index.

This approach dramatically reduces the number of documents that need to be scored, resulting in significant performance improvements while maintaining high recall accuracy.
# Key benefits

SEISMIC offers several advantages over traditional neural sparse search approaches:

- **Significant performance improvement**: Achieves at least 3x query speed improvement compared to two-phase queries under ≥90% recall conditions
- **Scalability**: Maintains consistent query performance even as datasets scale to billions of vectors
- **Memory efficiency**: Uses optimized caching strategies and optional quantization to manage memory usage
- **Backward compatibility**: Seamlessly integrates with existing neural sparse search infrastructure
- **Hybrid approach**: Automatically selects the optimal indexing strategy based on segment size
- **High searsch flexibility**: Users can smoothly tune the trade-off between high recall and low latency with the help of cut & heap_factor these two parameters.

## Configuration parameters

SEISMIC provides several tunable parameters to optimize performance for different use cases:

### Index-level settings

- **index.sparse**: Boolean flag to enable sparse ANN fields in the index
- **approximate_threshold**: Document count threshold that triggers SEISMIC algorithm on a segment (default: 100,000)

### Field mapping parameters

- **n_postings** (λ): Number of top documents to retain for each posting list
- **cluster_ratio**: Ratio used to determine cluster count in posting lists (default: 0.1)
- **summary_prune_ratio** (α): Ratio of tokens to keep in cluster summary vectors (default: 0.4)

### Query parameters

- **top_n**: Number of most important query tokens to retain (default: 10)
- **k**: Number of nearest neighbors to return
- **heap_factor**: Tuning parameter for recall vs. QPS trade-off (default: 1.0)
- **filter**: Optional boolean filter for pre-filtering or post-filtering

## Performance characteristics

SEISMIC is designed to excel in large-scale scenarios:

- **Query performance**: 3x+ improvement over two-phase queries with ≥90% recall
- **Memory usage**: Configurable caching strategies with circuit breakers to prevent resource exhaustion
- **Indexing overhead**: Minimal impact on indexing performance through hybrid approach
- **Scalability**: Linear performance scaling with dataset size## Fi
ltering support

SEISMIC supports both pre-filtering and post-filtering approaches:

### Post-filtering
Users can combine SEISMIC queries with boolean filters in compound queries. The algorithm first finds the top K nearest documents, then applies filters to produce final results.

### Pre-filtering
For efficient pre-filtering, SEISMIC first applies filter logic. If the filtered result size is smaller than K, it runs exact matching on filtered results. For larger filtered sets, it runs the SEISMIC algorithm with post-filtering.

## Memory management

SEISMIC implements sophisticated memory management strategies:

- **Flexible caching**: Configurable strategies allowing users to balance query performance against memory usage
- **Circuit breakers**: Prevent memory exhaustion and service degradation
- **Quantization support**: Multiple quantization levels to reduce memory footprint
- **Cache key management**: Uses IndexKey (combination of SegmentInfo and field name) for unique identification

## When to use SEISMIC

SEISMIC is particularly beneficial for:

- **Large-scale applications**: Datasets with millions to billions of documents where query performance is critical
- **High-throughput scenarios**: Applications requiring fast response times under heavy query loads
- **Memory-constrained environments**: Systems where traditional dense vector approaches are not feasible
- **Hybrid search systems**: Applications combining multiple retrieval methods for optimal performance

Consider SEISMIC when you need the efficiency of sparse retrieval but require better performance than traditional neural sparse search methods can provide at scale.

## Limitations

- **Index compatibility**: Currently, indices can only use either sparse ANN fields or k-NN fields, not both (limitation may be addressed in future versions)
- **Memory requirements**: While optimized, SEISMIC still requires significant memory for forward index and clustered posting lists
- **Complexity**: Additional configuration parameters require tuning for optimal performance

## Getting started

To implement SEISMIC in your OpenSearch cluster:

1. **Enable sparse indexing**: Set `index.sparse: true` in your index settings
2. **Configure field mappings**: Define sparse ANN field mappings with appropriate SEISMIC parameters
3. **Tune parameters**: Adjust clustering and query parameters based on your dataset characteristics
4. **Monitor performance**: Use OpenSearch monitoring tools to track query performance and memory usage

For detailed setup instructions, see [SEISMIC configuration]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration/).

## Next steps

- [Configure SEISMIC]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-configuration/) for detailed setup instructions
- [Explore SEISMIC examples]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-seismic-examples/) for practical implementation guidance
- Learn about [neural sparse search]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-search/) fundamentals
- Discover [hybrid search]({{site.url}}{{site.baseurl}}/vector-search/ai-search/hybrid-search/) techniques

## Further reading

- [Original SEISMIC paper](https://arxiv.org/abs/2404.18812): "Efficient Inverted Indexes for Approximate Retrieval over Learned Sparse Representations"
- [OpenSearch neural sparse search blog](https://opensearch.org/blog/improving-document-retrieval-with-sparse-semantic-encoders/): Learn about sparse encoding fundamentals
- [Two-phase query optimization](https://opensearch.org/blog/introducing-a-neural-sparse-two-phase-algorithm/): Understanding the foundation SEISMIC builds upon
- [Neural sparse search documentation]({{site.url}}{{site.baseurl}}/vector-search/ai-search/neural-sparse-search/): Core concepts and implementation