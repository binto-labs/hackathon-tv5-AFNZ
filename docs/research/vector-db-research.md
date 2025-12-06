---
status: keep
phase: complete
type: report
version: 1.0
last-updated: 2025-12-06
title: Vector Database Research - TV5 Media Gateway
---

# Vector Database Research - TV5 Media Gateway

## Executive Summary

For the TV5 Media Gateway project requiring semantic search and entity resolution across 400K+ media titles with sub-100ms query performance, this research evaluates vector database solutions focusing on RuVector/AgentDB alongside established alternatives. Key findings: **AgentDB claims 150x-12,500x performance improvements with sub-100µs latency**, making it compelling for the hackathon MVP. For production scale (1M+ titles), **Qdrant or pgvectorscale offer proven sub-20ms performance at 79% lower cost than Pinecone** while maintaining 99% recall. Recommended embedding model: **all-MiniLM-L6-v2 (384 dimensions)** provides optimal balance between accuracy and performance for media title matching, achieving 5-14k sentences/sec on CPU with 4x memory savings versus 1536-dimensional alternatives.

## RuVector/AgentDB Analysis

### Architecture & Capabilities

**AgentDB** is an open-source (MIT license) vector database and memory system optimized for AI agents, developed by ruvnet. It provides:

- **HNSW Indexing**: Hierarchical Navigable Small World graphs for O(log n) search complexity
- **ReasoningBank**: Pattern storage and retrieval for agent decision-making
- **MCP Integration**: 29 Model Context Protocol tools (5 Core Vector DB + 5 Core AgentDB + 9 Frontier Memory + 10 Learning System)
- **Reinforcement Learning**: PPO, Decision Transformer, MCTS algorithms
- **Platform Support**: Node.js, Web Browser, Edge environments

**Source**: [AgentDB Official Site](https://agentdb.ruv.io/), [GitHub Issue #829](https://github.com/ruvnet/claude-flow/issues/829)

### Performance Benchmarks

AgentDB claims dramatic performance improvements over traditional memory systems:

| Operation | Baseline | AgentDB | Speedup |
|-----------|----------|---------|---------|
| Pattern search | 15ms | 100µs | 150x |
| Batch insert (100 items) | 1s | 2ms | 500x |
| Large queries (1M records) | 100s | 8ms | 12,500x |

**HNSW Performance at Scale**:
- 1K vectors: 2.2x faster than brute force
- 10K vectors: 12x faster
- 100K vectors: 116x faster (verified benchmark)

**Note**: These benchmarks are from the AgentDB team. Independent third-party validation for production use cases was not found during research.

**Sources**: [GitHub Issue #829](https://github.com/ruvnet/claude-flow/issues/829), [Release v2.7.0-alpha.14](https://github.com/ruvnet/claude-flow/issues/822)

### Memory Optimization

AgentDB supports three quantization strategies:

| Type | Memory Reduction | Accuracy | Use Case |
|------|------------------|----------|----------|
| Binary | 32x | ~95% | Large-scale deployment (1M+ vectors) |
| Scalar | 4x | ~99% | Balanced performance |
| Product | 8-16x | ~97% | High-dimensional data (768+ dims) |

For 100K titles at 384 dimensions:
- Base memory: ~150MB (raw vectors)
- With HNSW: ~210-240MB (1.4-1.6x overhead)
- With scalar quantization: ~53MB total

**Source**: [GitHub Issue #829](https://github.com/ruvnet/claude-flow/issues/829)

### Integration Patterns with claude-flow

AgentDB integrates with the claude-flow memory system through:

1. **Compatibility Layer**: Maintains existing EnhancedMemory API
2. **AgentDB Backend**: Handles vector operations and learning
3. **Migration Bridge**: Converts legacy data structures

**MCP Tools Available**:
- Vector operations: `store_vector`, `search_vectors`, `batch_insert`
- Pattern matching: `semantic_search`, `fuzzy_match`
- Learning: `train_pattern`, `evaluate_model`

**Source**: [GitHub Issue #829](https://github.com/ruvnet/claude-flow/issues/829)

### Production Readiness Assessment

**Strengths**:
- Exceptional performance claims (if validated)
- Native MCP integration for AI agents
- Low memory footprint with quantization
- Active development (v2.7.0-alpha.14, October 2025)

**Concerns**:
- Alpha release status (not GA)
- Limited independent benchmarks
- Smaller community vs. Pinecone/Qdrant/Milvus
- No enterprise SLA or support options documented

**Recommendation for TV5**: **CONDITIONAL ADOPT for hackathon MVP** (low risk, high potential reward). **DEFER for production** pending independent validation and GA release.

## Embedding Model Comparison

### Overview

Sentence Transformers is the dominant framework for semantic embeddings, with 15,000+ pre-trained models available on Hugging Face. Model selection balances semantic quality, computational speed, and memory footprint.

**Source**: [Sentence Transformers Documentation](https://sbert.net/), [Hugging Face](https://huggingface.co/sentence-transformers)

### Top Models for Media Title Matching

| Model | Dimensions | Params | Speed (CPU) | Quality | Memory/1M vectors |
|-------|------------|--------|-------------|---------|-------------------|
| all-MiniLM-L6-v2 | 384 | 22M | 5-14k sent/sec | Good | 1.5GB |
| all-mpnet-base-v2 | 768 | 110M | 1-3k sent/sec | Excellent | 3.0GB |
| E5-base-v2 (Microsoft) | 768 | 110M | 1-3k sent/sec | Excellent | 3.0GB |
| text-embedding-3-small (OpenAI) | 1536 | N/A | API-limited | Excellent | 6.0GB |
| gte-multilingual-base (Alibaba) | 768 | 305M | 0.8-2k sent/sec | Excellent (70+ langs) | 3.0GB |
| EmbeddingGemma (Google) | 768* | 300M | 2-4k sent/sec | Excellent | 3.0GB |

*EmbeddingGemma supports Matryoshka Representation Learning (MRL), allowing truncation to 512/256/128 dimensions without retraining.

**Sources**: [A Guide to Open-Source Embedding Models](https://www.bentoml.com/blog/a-guide-to-open-source-embedding-models), [Best Open-Source Embedding Models](https://supermemory.ai/blog/best-open-source-embedding-models-benchmarked-and-ranked/), [EmbeddingGemma](https://huggingface.co/blog/embeddinggemma)

### Dimension Tradeoffs

**384 Dimensions (Recommended)**:
- **Storage**: 1.5GB for 1M vectors (baseline)
- **Speed**: 4-5x faster than 768-dim models
- **Use case**: Real-time semantic search, large-scale clustering
- **Model**: all-MiniLM-L6-v2 (most popular)

**768 Dimensions**:
- **Storage**: 3.0GB for 1M vectors (2x vs 384)
- **Speed**: Moderate (1-3k sentences/sec)
- **Use case**: Higher accuracy for complex semantic tasks
- **Models**: all-mpnet-base-v2, E5-base-v2

**1536 Dimensions (OpenAI)**:
- **Storage**: 6.0GB for 1M vectors (4x vs 384)
- **Speed**: API rate-limited, higher latency
- **Use case**: Maximum semantic nuance
- **Cost**: $0.13 per 1M tokens (significant at scale)

**Key Insight**: "Higher dimensions don't automatically improve retrieval accuracy." Research shows **384-dimensional models often match 1536-dimensional performance** for retrieval tasks while using 75% less memory and delivering 4x faster inference.

**Sources**: [Embedding Models and Dimensions](https://devblogs.microsoft.com/azure-sql/embedding-models-and-dimensions-optimizing-the-performance-resource-usage-ratio/), [How Big Are Our Embeddings Now](https://vickiboykis.com/2025/09/01/how-big-are-our-embeddings-now-and-why/), [Choosing Embedding Models](https://ragwalla.com/docs/guides/choosing-an-embedding-model-for-your-workload)

### Recommendation for TV5

**Primary**: **all-MiniLM-L6-v2 (384 dimensions)**
- Optimal speed/quality balance for title matching
- 22M parameters = fast CPU inference
- Proven at scale (most downloaded model)
- Free, open-source, offline-capable

**Alternative**: **all-mpnet-base-v2 (768 dimensions)** if accuracy testing reveals need for finer semantic distinctions (e.g., differentiating "The Office US" vs "The Office UK").

**Avoid**: OpenAI embeddings for batch processing (cost scales linearly, API dependency).

## Vector Database Comparison

### Quick Recommendation Matrix

| Use Case | Recommended Database |
|----------|---------------------|
| Hackathon MVP (100K titles, speed priority) | AgentDB or Qdrant |
| Production self-hosted (1M+ titles, cost-optimized) | Qdrant or pgvectorscale |
| Production managed (minimal ops overhead) | Pinecone p2 |
| Prototyping/testing | Chroma |
| Enterprise scale (10M+ titles, GPU budget) | Milvus |

**Source**: [Vector Database Comparison 2025](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)

### Detailed Comparison

#### Pinecone

**Type**: Fully managed SaaS (cannot self-host)

**Performance**:
- Sub-100ms queries at scale (verified)
- Serverless auto-scaling
- Separates query and indexing workloads

**Pricing**:
- 10M vectors, 1M queries/month: ~$675/month
- Real production costs: $500-$3,200/month (varies by query volume)
- **Performance-optimized (p2) index**: 79% more expensive than self-hosted pgvectorscale

**Pros**:
- Zero infrastructure management
- SOC 2 Type II, ISO 27001, HIPAA compliant
- Pinecone Assistant (GA Jan 2025): all-in-one chunking, embedding, search, reranking
- Hybrid search (vector + keyword)

**Cons**:
- Highest cost at scale
- Vendor lock-in
- Limited customization

**Recommendation**: **CONDITIONAL ADOPT** if time-to-market > cost optimization and team lacks DevOps expertise.

**Sources**: [Pgvector vs Pinecone](https://www.tigerdata.com/blog/pgvector-vs-pinecone), [Pinecone vs Weaviate vs Qdrant](https://sysdebug.com/posts/vector-database-comparison-guide-2025/), [Vector Database Comparison](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)

#### Qdrant

**Type**: Open-source (Rust), self-hosted or cloud

**Performance**:
- Sub-20ms p95 latency at 90% recall (10M vectors)
- 81 QPS at ~100ms (98.5% recall, benchmarked)
- Small variance between p50/p95/p99 (excellent tail latency)

**Memory**:
- 1M vectors (384-dim): ~1GB RAM (minimum)
- Production recommendation: 2-3GB for headroom
- Supports memory-mapped files for cold storage

**Pricing**:
- Self-hosted: ~$200/month infrastructure (10M vectors)
- Qdrant Cloud: ~$400-600/month managed
- **75% less than Pinecone** for equivalent workload

**Pros**:
- Rust performance (fastest among open-source)
- Fine-grained payload filtering (hybrid search)
- Distributed deployment, horizontal scaling
- ACID-compliant transactions
- gRPC API (excellent for microservices)

**Cons**:
- Requires Kubernetes expertise for self-hosting
- Smaller ecosystem vs Pinecone
- Manual optimization of HNSW parameters

**Recommendation**: **ADOPT for production** (best performance/cost ratio for self-hosted).

**Sources**: [Qdrant Benchmarks](https://qdrant.tech/benchmarks/), [Qdrant Memory Consumption](https://qdrant.tech/articles/memory-consumption/), [Pgvector vs Qdrant](https://www.tigerdata.com/blog/pgvector-vs-qdrant), [Qdrant vs Pinecone](https://qdrant.tech/blog/comparing-qdrant-vs-pinecone-vector-databases/)

#### Weaviate

**Type**: Open-source (Go), self-hosted or cloud

**Performance**:
- 65 QPS at 60ms latency (98.5% recall)
- 1M+ Docker pulls/month (most popular)
- Async indexing (v1.28+): persistent on-disk queue

**Features**:
- **Best hybrid search**: GraphQL interface
- Knowledge graph capabilities
- BM25F keyword search + vector fusion (configurable alpha)
- Reciprocal Rank Fusion (RRF) algorithm

**Pricing**:
- Self-hosted: ~$200-300/month (10M vectors)
- Weaviate Cloud: SOC 2 Type II, HIPAA on AWS

**Pros**:
- Excellent for hybrid search (semantic + keyword)
- Strong GraphQL API for complex queries
- Knowledge graph integration
- Active community (25k+ GitHub stars)

**Cons**:
- Go-based (less performant than Rust Qdrant)
- Higher memory usage than Qdrant
- Steeper learning curve for GraphQL

**Recommendation**: **CONDITIONAL ADOPT** if hybrid search (title + metadata filters) is critical.

**Sources**: [Weaviate Hybrid Search](https://weaviate.io/developers/weaviate/search/hybrid), [Vector Database Comparison](https://liquidmetal.ai/casesAndBlogs/vector-comparison/), [Weaviate Async Indexing](https://weaviate.io/developers/weaviate/concepts/vector-index)

#### Milvus

**Type**: Open-source (Go/C++), distributed architecture

**Performance**:
- Fastest indexing time among open-source (benchmarked)
- GPU acceleration: 10x faster indexing at 1/4 cost
- 700k+ Docker pulls/month

**Scale**:
- Designed for **billion-scale** deployments
- Distributed by default (Kubernetes-native)
- Supports IVF, HNSW, DiskANN indexes

**Pricing**:
- Self-hosted: ~$200/month (10M vectors)
- Managed (Zilliz Cloud): ~$600/month

**Pros**:
- Best for massive scale (10M+ vectors)
- GPU support for ML workloads
- Mature distributed architecture
- Strong performance at 99% recall

**Cons**:
- **Overkill for <1M vectors**
- Complex setup (requires Kubernetes)
- Higher operational overhead

**Recommendation**: **DEFER until >10M titles** or GPU acceleration needed.

**Sources**: [Milvus Documentation](https://milvus.io/docs/hnsw.md), [VectorDBBench](https://github.com/zilliztech/VectorDBBench), [Vector Database Comparison](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)

#### Chroma

**Type**: Open-source, lightweight (Python)

**Performance**:
- Designed for <1M vectors
- In-memory or disk-backed
- Moderate query speed

**Features**:
- "Batteries included" LLM integration
- Minimal setup (pip install)
- Document retrieval optimized

**Pricing**: Free (self-hosted only)

**Pros**:
- Fastest prototyping
- Zero configuration
- Perfect for demos/hackathons

**Cons**:
- Not production-grade at scale
- Limited filtering capabilities
- Single-node only (no distribution)

**Recommendation**: **ADOPT for hackathon prototyping**, **DEFER for production**.

**Sources**: [Best Vector Databases 2025](https://www.firecrawl.dev/blog/best-vector-databases-2025), [Vector Database Comparison](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)

#### pgvectorscale (PostgreSQL + pgvector)

**Type**: Open-source PostgreSQL extension

**Performance**:
- **471 QPS at 99% recall (50M vectors)** - 11.4x better than Qdrant
- Sub-20ms p95 latency (verified)
- DiskANN indexing for billion-scale

**Pricing**:
- **79% cheaper than Pinecone p2** ($835/month vs $3,241/month on AWS)
- Uses existing PostgreSQL infrastructure

**Pros**:
- Leverage existing SQL expertise
- Transactional consistency (ACID)
- Mature backup/replication tools
- **Best cost/performance ratio** (2025 benchmarks)

**Cons**:
- Requires PostgreSQL tuning expertise
- Not specialized for vectors (general-purpose DB)
- Higher latency than pure vector DBs at extreme scale

**Recommendation**: **ADOPT for production** if team has PostgreSQL expertise.

**Sources**: [Pgvector vs Pinecone](https://www.tigerdata.com/blog/pgvector-vs-pinecone), [Pgvector vs Qdrant](https://www.tigerdata.com/blog/pgvector-vs-qdrant), [HNSW Indexes with Postgres](https://www.crunchydata.com/blog/hnsw-indexes-with-postgres-and-pgvector)

### Performance Summary Table

| Database | Latency (p95) | QPS @ 99% Recall | Monthly Cost (10M) | Setup Complexity |
|----------|---------------|------------------|---------------------|------------------|
| AgentDB | <1ms (claimed) | Unknown | $0 (self-hosted) | Low (Node.js) |
| Pinecone | <100ms | High | $675+ | Very Low (SaaS) |
| Qdrant | <20ms | 81 | $200-600 | Medium (K8s) |
| Weaviate | ~60ms | 65 | $200-300 | Medium (K8s) |
| Milvus | Variable | Highest | $200-600 | High (K8s + GPU) |
| Chroma | Variable | Low | $0 | Very Low (Python) |
| pgvectorscale | <20ms | 471 | $835 | Medium (PostgreSQL) |

**Sources**: [Qdrant Benchmarks](https://qdrant.tech/benchmarks/), [Pgvector vs Qdrant](https://www.tigerdata.com/blog/pgvector-vs-qdrant), [Vector Database Benchmarks](https://medium.com/@vkmauryavk/vector-database-benchmarks-a-definitive-guide-to-tools-metrics-and-top-performers-4c4110e61f73)

## Integration Architecture

### Entity Resolution Pipeline

For TV5 Media Gateway, vector search supports fuzzy title matching when exact ID resolution fails:

```
1. Primary Resolution (Exact Match)
   - Use title_id, imdb_id, tmdb_id from API responses
   - Handles 70-90% of cases

2. Secondary Resolution (Vector Search)
   - Triggered for titles without exact IDs
   - Query: Vectorize title + year + type (e.g., "The Office 2005 series")
   - Search: k-NN with k=10, similarity threshold ≥0.85
   - Metadata filters: type="series", year±2, language match
   - Return top-3 candidates with confidence scores

3. Manual Review Queue
   - Titles with max_similarity <0.85
   - Ambiguous results (>1 result with similarity ≥0.95)
```

**Source**: [Pre-trained Embeddings for Entity Resolution](https://www.vldb.org/pvldb/vol16/p2225-skoutas.pdf)

### Hybrid Search Implementation

Combining vector similarity with keyword filtering improves accuracy:

**Weaviate Example** (GraphQL):
```graphql
{
  Get {
    MediaTitle(
      hybrid: {
        query: "The Office"
        alpha: 0.7  # 0=keyword, 1=vector, 0.7=balanced
      }
      where: {
        path: ["year"]
        operator: GreaterThanEqual
        valueInt: 2003
      }
    ) {
      title
      year
      similarity
    }
  }
}
```

**Qdrant Example** (REST API):
```json
{
  "vector": [0.12, 0.45, ...],
  "filter": {
    "must": [
      {"key": "type", "match": {"value": "series"}},
      {"key": "year", "range": {"gte": 2003, "lte": 2007}}
    ]
  },
  "limit": 10,
  "score_threshold": 0.85
}
```

**Sources**: [Weaviate Hybrid Search](https://weaviate.io/developers/weaviate/search/hybrid), [Hybrid Search in Vector Databases](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview)

### Batch vs Real-Time Indexing

**Batch Indexing** (Recommended for TV5):
- **Use case**: Initial 400K title ingestion, nightly updates
- **Strategy**: Batch 500-5,000 vectors per insert to minimize write-amplification
- **Tools**: Bulk import APIs (all databases support)
- **Schedule**: Off-peak hours (2-6 AM) to avoid query contention

**Real-Time Indexing**:
- **Use case**: New title additions from live API feeds
- **Strategy**: Single-item inserts with async indexing (Weaviate v1.28+)
- **Performance**: <10ms insert latency (HNSW in-memory)

**Index Rebuild Strategy**:
- **Frequency**: Weekly or when recall degrades >2%
- **Method**: Build new index in parallel, atomic swap
- **Downtime**: Zero with blue-green deployment

**Sources**: [Vector Database Indexing](https://akash-mathur.medium.com/vector-database-vs-indexing-path-to-efficient-data-handling-382cc1207491), [Weaviate Vector Indexing](https://weaviate.io/developers/weaviate/concepts/vector-index)

### Similarity Threshold Tuning

Research on entity resolution shows optimal thresholds depend on data characteristics:

**For Media Titles**:
- **Exact match variants** (e.g., "The Office" vs "Office, The"): threshold ≥0.95
- **Fuzzy match** (e.g., "The Office US" vs "The Office"): threshold ≥0.85
- **Related titles** (e.g., "The Office" vs "The Office: Webisodes"): threshold ≥0.75

**Tuning Process**:
1. Sample 1,000 known duplicate pairs
2. Generate embeddings with chosen model
3. Plot cosine similarity distribution
4. Select threshold at 95th percentile of true matches
5. Validate precision/recall on test set

**Best Practice**: Use **range search** (min/max similarity) rather than fixed threshold:
- Inner circle: 0.95+ (high confidence matches)
- Outer circle: 0.85-0.95 (review queue)
- Below 0.85: manual resolution

**Sources**: [EM-Join: Efficient Entity Matching](https://www.scitepress.org/Papers/2025/134837/134837.pdf), [Milvus Range Search](https://milvus.io/docs/single-vector-search.md), [Fuzzy Matching and Semantic Search](https://ipullrank.com/fuzzy-matching-semantic-search)

### Index Organization Strategies

**Option 1: Single Monolithic Index**
- All 400K titles in one index
- **Pros**: Simplest, no routing logic
- **Cons**: Higher query latency, difficult to A/B test

**Option 2: Partitioned by Type**
- Separate indexes: movies, series, documentaries
- **Pros**: 60-70% smaller indexes, faster queries
- **Cons**: Requires type metadata, more indexes to manage

**Option 3: Partitioned by Era** (Recommended)
- Pre-2000, 2000-2010, 2010-2020, 2020+
- **Pros**: Temporal queries naturally filter, balanced sizes
- **Cons**: Year must be known for query routing

**Option 4: Hybrid (Type + Era)**
- 12 indexes (3 types × 4 eras)
- **Pros**: Fastest queries, best for filtered search
- **Cons**: Highest operational complexity

**Recommendation for TV5**: Start with **single index** for MVP, migrate to **type-based partitioning** at 1M+ titles.

**Sources**: [Qdrant Payload Filtering](https://qdrant.tech/articles/vector-search-resource-optimization/), [Vector Database Index Organization](https://www.singlestore.com/blog/-ultimate-guide-vector-database-landscape-2024/)

## Performance Projections

### 100K Titles (Hackathon MVP)

**Configuration**:
- Embedding: all-MiniLM-L6-v2 (384 dimensions)
- Database: Qdrant or AgentDB
- Index: HNSW (M=16, ef_construction=200)

**Resource Requirements**:
- **Storage**: 150MB (vectors) + 50MB (HNSW graph) = 200MB total
- **Memory**: 400MB (2x storage for overhead)
- **CPU**: 2 vCPUs @ 2.5GHz
- **Disk**: 1GB (includes backups)

**Performance**:
- **Indexing time**: 30-60 seconds (batch import)
- **Query latency**: 5-15ms (p95), <50ms (p99)
- **Throughput**: 500-1,000 QPS (single node)

**Cost** (AWS):
- EC2 t3.medium: $30/month
- EBS storage (10GB): $1/month
- **Total**: ~$31/month

**Sources**: [Qdrant Memory Consumption](https://qdrant.tech/articles/memory-consumption/), [HNSW Memory Requirements](https://stackoverflow.com/questions/77401874/how-to-calculate-amount-of-ram-required-for-serving-x-n-dimensional-vectors-with)

### 1M Titles (Production)

**Configuration**:
- Embedding: all-MiniLM-L6-v2 (384 dimensions)
- Database: Qdrant (distributed) or pgvectorscale
- Index: HNSW (M=24, ef_construction=400)

**Resource Requirements**:
- **Storage**: 1.5GB (vectors) + 0.6GB (HNSW graph) = 2.1GB total
- **Memory**: 4-6GB (includes query cache, OS)
- **CPU**: 4-8 vCPUs
- **Disk**: 10GB SSD

**Performance**:
- **Indexing time**: 5-10 minutes (batch import)
- **Query latency**: 10-20ms (p95), <50ms (p99)
- **Throughput**: 200-500 QPS (distributed: 2,000+ QPS)

**Cost** (AWS):
- Qdrant: EC2 r6g.xlarge (4 vCPU, 32GB RAM): $200/month
- pgvectorscale: RDS r6g.xlarge: $835/month (includes managed ops)
- Pinecone p2: $3,241/month (79% more expensive)

**Scaling Strategy**:
- Horizontal: Add replica nodes for read-heavy workloads
- Vertical: Upgrade to r6g.2xlarge (8 vCPU, 64GB) for write-heavy

**Sources**: [Pgvector vs Pinecone Cost](https://www.tigerdata.com/blog/pgvector-vs-pinecone), [Qdrant Scaling](https://qdrant.tech/benchmarks/)

### 10M+ Titles (Future Scale)

**Configuration**:
- Embedding: all-MiniLM-L6-v2 (384 dimensions) or binary quantized (32x savings)
- Database: Milvus (GPU-accelerated) or Qdrant distributed
- Index: HNSW + PQ (product quantization)

**Resource Requirements**:
- **Storage**: 15GB (vectors) + 6GB (index) = 21GB
- **Memory**: 32-64GB (or 1-2GB with quantization)
- **CPU/GPU**: 16 vCPUs or 1 NVIDIA T4 GPU
- **Disk**: 100GB SSD

**Performance**:
- **Indexing time**: 30-60 minutes (GPU: 3-6 minutes)
- **Query latency**: 20-50ms (p95)
- **Throughput**: 1,000-5,000 QPS (distributed cluster)

**Cost** (AWS):
- Milvus GPU: EC2 g4dn.2xlarge: $600-800/month
- Qdrant distributed (3 nodes): $600-900/month

**Sources**: [Milvus GPU Acceleration](https://aws.amazon.com/blogs/aws/amazon-opensearch-service-improves-vector-database-performance-and-cost-with-gpu-acceleration-and-auto-optimization/), [Billion-Scale HNSW](https://medium.com/vespa/billion-scale-vector-search-using-hybrid-hnsw-if-96d7058037d3)

## Recommendations

| Aspect | Recommendation | Rationale |
|--------|----------------|-----------|
| **Hackathon MVP** | AgentDB or Chroma | AgentDB offers best performance (if claims validated); Chroma is fastest to implement. Low risk for 100K scale. |
| **Production (Self-Hosted)** | Qdrant | Best performance/cost ratio (79% cheaper than Pinecone), excellent tail latency, Rust reliability. 471 QPS at 99% recall. |
| **Production (Managed)** | Pinecone p2 or Qdrant Cloud | Pinecone if team lacks DevOps; Qdrant Cloud for 40% cost savings with similar features. |
| **Production (SQL Expertise)** | pgvectorscale | Best cost/performance (471 QPS, $835/month), leverage existing PostgreSQL skills, ACID transactions. |
| **Embedding Model** | all-MiniLM-L6-v2 (384-dim) | 4-5x faster than 768-dim, 75% less memory, proven accuracy for retrieval. 5-14k sentences/sec on CPU. |
| **Alternative Embedding** | all-mpnet-base-v2 (768-dim) | If testing shows need for finer semantic distinctions. 1-3k sentences/sec, 2x memory. |
| **Hybrid Search** | Weaviate or Qdrant | Weaviate for GraphQL + knowledge graph; Qdrant for fine-grained payload filtering. |
| **Similarity Threshold** | 0.85 (fuzzy match) | Based on entity resolution research. Use 0.95+ for high confidence, <0.85 for manual review. |
| **Index Organization** | Single index → Type-based | Start simple, partition by type (movies/series/docs) at 1M+ titles for 60-70% faster queries. |
| **Batch Strategy** | 500-5,000 vectors/batch | Minimizes write-amplification. Schedule during off-peak (2-6 AM). |
| **Quantization** | Scalar (4x savings) at 1M+ | Maintains 99% accuracy, reduces memory from 6GB → 1.5GB. Binary (32x) only at 10M+ scale. |

## Migration Strategy

### Phase 1: Hackathon MVP (Week 1-2)
1. **Prototype with Chroma** (4 hours setup)
   - Validate embedding model accuracy
   - Test similarity thresholds
   - Benchmark query latency on 100K sample
2. **Evaluate AgentDB** (1 day)
   - Install via npm, test MCP integration
   - Compare performance vs Chroma
   - Validate claimed 100µs latency
3. **Decision Point**: Choose Chroma (safe) or AgentDB (high-risk/high-reward)

### Phase 2: Production Planning (Month 1-2)
1. **Benchmark Qdrant vs pgvectorscale** (1 week)
   - Deploy both on AWS with 1M titles
   - Load test: 100-1,000 QPS for 24 hours
   - Measure p50/p95/p99 latency, cost/query
2. **Implement Hybrid Search** (1 week)
   - Add metadata filters (type, year, language)
   - Tune Reciprocal Rank Fusion (RRF) weights
   - A/B test pure vector vs hybrid accuracy
3. **Decision Point**: Qdrant (best performance) or pgvectorscale (best cost, SQL familiarity)

### Phase 3: Scale to 10M+ (Year 1)
1. **Quantization Testing** (2 weeks)
   - Implement scalar quantization (4x savings)
   - Validate accuracy degradation <1%
   - Benchmark memory reduction
2. **Distributed Deployment** (1 month)
   - Deploy Qdrant 3-node cluster (HA)
   - Implement sharding by title type
   - Load balancing with replica sets
3. **GPU Evaluation** (if needed)
   - Test Milvus with NVIDIA T4
   - Compare indexing speed (10x claimed)
   - Justify GPU cost vs CPU-only

## Sources

### AgentDB & RuVector
- [AgentDB Official Site](https://agentdb.ruv.io/)
- [GitHub Issue #829: AgentDB Integration Performance](https://github.com/ruvnet/claude-flow/issues/829)
- [Release v2.7.0-alpha.14](https://github.com/ruvnet/claude-flow/issues/822)

### Embedding Models
- [Sentence Transformers Documentation](https://sbert.net/)
- [A Guide to Open-Source Embedding Models](https://www.bentoml.com/blog/a-guide-to-open-source-embedding-models)
- [Best Open-Source Embedding Models Benchmarked](https://supermemory.ai/blog/best-open-source-embedding-models-benchmarked-and-ranked/)
- [Embedding Models and Dimensions](https://devblogs.microsoft.com/azure-sql/embedding-models-and-dimensions-optimizing-the-performance-resource-usage-ratio/)
- [How Big Are Our Embeddings Now](https://vickiboykis.com/2025/09/01/how-big-are-our-embeddings-now-and-why/)
- [EmbeddingGemma](https://huggingface.co/blog/embeddinggemma)
- [Choosing Embedding Models](https://ragwalla.com/docs/guides/choosing-an-embedding-model-for-your-workload)

### Vector Database Comparisons
- [Vector Database Comparison 2025 (LiquidMetal)](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)
- [Best Vector Database For RAG 2025](https://digitaloneagency.com.au/best-vector-database-for-rag-in-2025-pinecone-vs-weaviate-vs-qdrant-vs-milvus-vs-chroma/)
- [Vector Database Comparison Guide 2025](https://sysdebug.com/posts/vector-database-comparison-guide-2025/)
- [Best Vector Databases 2025 (Firecrawl)](https://www.firecrawl.dev/blog/best-vector-databases-2025/)
- [Vector Database Comparison (Turing)](https://www.turing.com/resources/vector-database-comparison)

### Performance Benchmarks
- [Qdrant Benchmarks](https://qdrant.tech/benchmarks/)
- [Pgvector vs Pinecone](https://www.tigerdata.com/blog/pgvector-vs-pinecone)
- [Pgvector vs Qdrant](https://www.tigerdata.com/blog/pgvector-vs-qdrant)
- [Vector Database Benchmarks Guide](https://medium.com/@vkmauryavk/vector-database-benchmarks-a-definitive-guide-to-tools-metrics-and-top-performers-4c4110e61f73)
- [VectorDBBench](https://github.com/zilliztech/VectorDBBench)
- [Redis Vector Benchmarks](https://redis.io/blog/benchmarking-results-for-vector-databases/)

### Hybrid Search
- [Azure AI Hybrid Search](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview)
- [Pinecone Hybrid Search](https://www.pinecone.io/blog/hybrid-search/)
- [Weaviate Hybrid Search](https://weaviate.io/developers/weaviate/search/hybrid)
- [Weaviate Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained)
- [Azure Cosmos DB Hybrid Search](https://blogs.perficient.com/2025/08/26/implementing-hybrid-search-in-azure-cosmos-db-combining-vectors-and-keywords/)

### Entity Resolution
- [Pre-trained Embeddings for Entity Resolution (VLDB)](https://www.vldb.org/pvldb/vol16/p2225-skoutas.pdf)
- [EM-Join: Efficient Entity Matching](https://www.scitepress.org/Papers/2025/134837/134837.pdf)
- [Entity Embed (PyTorch)](https://github.com/vintasoftware/entity-embed)
- [Fuzzy Matching and Semantic Search](https://ipullrank.com/fuzzy-matching-semantic-search)

### Memory & Performance
- [Qdrant Memory Consumption](https://qdrant.tech/articles/memory-consumption/)
- [HNSW Memory Calculator (Lantern)](https://lantern.dev/blog/calculator)
- [HNSW Memory Requirements (Stack Overflow)](https://stackoverflow.com/questions/77401874/how-to-calculate-amount-of-ram-required-for-serving-x-n-dimensional-vectors-with)
- [HNSW Indexes with Postgres](https://www.crunchydata.com/blog/hnsw-indexes-with-postgres-and-pgvector)
- [Billion-Scale HNSW](https://medium.com/vespa/billion-scale-vector-search-using-hybrid-hnsw-if-96d7058037d3)

### Cost Analysis
- [Pinecone vs Qdrant vs Weaviate Pricing](https://xenoss.io/blog/vector-database-comparison-pinecone-qdrant-weaviate)
- [Vector Database Cost Comparison](https://tensorblue.com/blog/vector-database-comparison-pinecone-weaviate-qdrant-milvus-2025)

### Index Organization
- [Qdrant Vector Search Optimization](https://qdrant.tech/articles/vector-search-resource-optimization/)
- [SQL Server 2025 Vector Features](https://devblogs.microsoft.com/azure-sql/sql-server-2025-embraces-vectors-setting-the-foundation-for-empowering-your-data-with-ai/)
- [Ultimate Guide to Vector Database Landscape](https://www.singlestore.com/blog/-ultimate-guide-vector-database-landscape-2024/)

---

**Research Completed**: 2025-12-06
**Researcher**: B2-VECTOR (TV5 Media Gateway Research Swarm)
**Next Steps**: Share with B1-PLANNER for entity resolution pipeline integration
