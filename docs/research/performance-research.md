---
status: keep
phase: complete
type: report
version: 1.0
last-updated: 2025-12-06
title: Performance Research - TV5 Media Gateway
---

# Performance Research - TV5 Media Gateway

## Executive Summary

For TV5 Media Gateway hackathon MVP targeting 100K titles scaling to 1M+ production, performance architecture must handle TMDB's 50 req/sec limit, deliver sub-200ms p95 API responses, and support 10-50 concurrent users. Key strategies: multi-layer caching (in-memory + Redis + pre-warming), token bucket rate limiting with priority queuing, and vector database optimization achieving 1,200+ QPS at 95% recall for semantic search.

## Scale Projections

### 100K Titles (Hackathon MVP)

| Component | Storage | Memory | QPS Target | Notes |
|-----------|---------|--------|------------|-------|
| **Raw Metadata** | ~500 MB | 250 MB | 20-50 | JSON documents averaging 5KB each |
| **Vector Embeddings (768D)** | ~293 MB | 440 MB | 50-100 | 3KB per vector × 100K × 1.5x index overhead[¹](#sources) |
| **Redis Cache** | ~1.5 GB | 2 GB | 500-1000 | Hot metadata + embeddings + availability |
| **PostgreSQL** | ~2 GB | 1 GB | 100-200 | Relational metadata with indices |
| **Total Estimate** | **~4.3 GB** | **~3.7 GB** | **100-200 QPS** | Single-server deployment viable |

### 1M Titles (Production Scale)

| Component | Storage | Memory | QPS Target | Notes |
|-----------|---------|--------|------------|-------|
| **Raw Metadata** | ~5 GB | 2.5 GB | 200-500 | Distributed across multiple nodes |
| **Vector Embeddings (768D)** | ~2.93 GB | 4.4 GB | 500-1200 | Binary quantization can reduce to 550 MB[²](#sources) |
| **Redis Cache** | ~15 GB | 20 GB | 5000-10000 | Distributed Redis cluster recommended |
| **PostgreSQL** | ~20 GB | 10 GB | 1000-2000 | Read replicas + connection pooling |
| **Total Estimate** | **~43 GB** | **~37 GB** | **1000-2000 QPS** | Multi-server architecture required |

### Storage Calculation Details

**Vector Embeddings** (768 dimensions):
- Per vector: 768 × 4 bytes = 3,072 bytes (~3 KB)
- 100K vectors: ~293 MB raw, ~440 MB with HNSW indexing overhead
- 1M vectors: ~2.93 GB raw, ~4.4 GB with indexing[³](#sources)

**Metadata Documents** (TMDB-based):
- Title metadata: ~2-3 KB (title, synopsis, release date, ratings)
- Cast/crew: ~1-2 KB (top 10 actors, directors)
- Streaming availability: ~500 bytes (platform IDs, regions, URLs)
- Total per title: ~4-5 KB average

### User Query Patterns

Based on streaming platform best practices:

| Query Type | Frequency | Cache Hit Target | Response Time Target |
|------------|-----------|------------------|---------------------|
| Browse/Discovery | 40% | 95% | <100ms (cached) |
| Search | 30% | 70% | <300ms (vector + cache) |
| Detail View | 20% | 90% | <150ms (cached) |
| Recommendations | 10% | 60% | <500ms (compute + cache) |

**Peak Load Estimation** (Hackathon Demo):
- Concurrent users: 10-50 judges/attendees
- Requests per user: 5-10 per minute (browsing)
- Peak QPS: 50 × 10 / 60 = **8-10 QPS**
- With 3x safety margin: **30 QPS target**

## Latency Targets

### API Response Time SLAs

| Operation | p50 | p95 | p99 | Priority | Rationale |
|-----------|-----|-----|-----|----------|-----------|
| **Cached Metadata Retrieval** | 20ms | 50ms | 100ms | Critical | In-memory/Redis lookup[⁴](#sources) |
| **Vector Similarity Search** | 50ms | 150ms | 300ms | High | 100K vectors HNSW index |
| **Multi-Source Aggregation** | 100ms | 300ms | 500ms | High | TMDB + JustWatch + cache |
| **Semantic Search (cold)** | 150ms | 400ms | 800ms | Medium | Vector search + metadata enrichment |
| **Availability Lookup** | 30ms | 100ms | 200ms | Critical | Time-sensitive streaming data |
| **Recommendation Generation** | 200ms | 600ms | 1000ms | Low | Complex multi-vector computation |

### Industry Benchmarks (2025)

- **User Experience Thresholds**: 100ms feels instant, 300ms is perceptible, 1000ms breaks flow[⁵](#sources)
- **SLA Best Practice**: Define percentile-based SLOs (e.g., "95% of requests <300ms over 30 days")[⁶](#sources)
- **Tail Latency Importance**: P95/P99 matter more than average—5% slow requests can affect high-value traffic[⁷](#sources)
- **Production Example**: Arcjet achieves 25ms p95 for security API, targeting 20-30ms range[⁸](#sources)

### Percentile Monitoring Strategy

```plaintext
P50 (Median)    → Detect broad performance regressions
P95 (5% tail)   → Primary SLO target, tune system performance
P99 (1% tail)   → Expose architectural bottlenecks & outliers
```

**Alert Configuration**:
- **Warning**: P95 > 200ms for 5 minutes
- **Critical**: P95 > 300ms for 2 minutes OR P99 > 1000ms
- **Error Budget**: Allow 5% of requests to exceed SLO over 30-day window

## Caching Architecture

### Multi-Layer Strategy

```plaintext
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
│                    (Rate Limiting + Routing)                │
└────────────────────────────┬────────────────────────────────┘
                             │
                ┌────────────┴─────────────┐
                │                          │
         ┌──────▼──────┐          ┌───────▼────────┐
         │  In-Memory  │          │  Redis Cache   │
         │   (L1)      │          │     (L2)       │
         │  100-500 MB │          │   1.5-15 GB    │
         │  <5ms       │          │   10-30ms      │
         └──────┬──────┘          └───────┬────────┘
                │                          │
                └────────────┬─────────────┘
                             │
                    ┌────────▼────────┐
                    │  PostgreSQL +   │
                    │  Vector DB      │
                    │  (Source)       │
                    │  50-200ms       │
                    └─────────────────┘
```

### Cache Layer Configuration

| Layer | Technology | TTL Strategy | Use Case | Size |
|-------|-----------|--------------|----------|------|
| **L1: In-Memory** | Node.js Map/LRU | 5-15 min | Hot metadata (top 1000 titles) | 100-500 MB |
| **L2: Redis** | Redis Cluster | Variable | All metadata + embeddings | 1.5-15 GB |
| **L3: Database** | PostgreSQL + Qdrant | N/A | Source of truth | 4-43 GB |

### TTL Patterns by Data Type

Based on 2025 caching best practices[⁹](#sources):

| Data Type | TTL | Invalidation Strategy | Jitter | Rationale |
|-----------|-----|----------------------|--------|-----------|
| **Title Metadata** | 24 hours | Event-driven on updates | ±10% | Rarely changes, TMDB updates infrequent |
| **Streaming Availability** | 1-6 hours | Event-driven + TTL hybrid | ±20% | Changes frequently, critical freshness |
| **Vector Embeddings** | 7 days | Manual invalidation only | N/A | Static once computed, expensive to regenerate |
| **Search Results** | 15 minutes | TTL only | ±5 min | User-specific, acceptable staleness |
| **Recommendations** | 30 minutes | Stale-while-revalidate | ±10 min | Personalized, background refresh OK |
| **Popular/Trending** | 5 minutes | TTL only | ±1 min | High traffic, needs freshness |

**TTL Jitter Pattern**: Add random time (e.g., ±20%) to prevent cache stampede when multiple keys expire simultaneously[¹⁰](#sources).

### Cache Invalidation Patterns

**Hybrid Approach** (2025 Best Practice)[¹¹](#sources):

1. **Event-Driven** (Critical Data):
   - Use for streaming availability changes
   - Webhook from data sources → invalidate Redis keys
   - Pub/sub pattern for distributed invalidation
   - Tools: Redis Pub/Sub, Kafka for event streaming

2. **Time-Based TTL** (Less Critical):
   - Use for metadata, search results, recommendations
   - Short TTL (5-15 min) for dynamic content
   - Long TTL (24 hours) for static content

3. **Stale-While-Revalidate** (User Experience):
   - Serve cached content while fetching fresh data in background
   - HTTP header: `Cache-Control: max-age=3600, stale-while-revalidate=86400`
   - Ideal for recommendations and personalized content

### Cache Warming Strategy

**Pre-Load Patterns**[¹²](#sources):

```bash
# Deployment-time warming (post-deploy hook)
curl -X POST /api/admin/cache/warm \
  --data '{"strategy": "popular", "limit": 1000}'

# Scheduled warming (nightly cron)
0 2 * * * /scripts/cache-warm.sh --preset demo-titles

# Sitemap-based crawling (for demo)
curl -X POST /api/admin/cache/warm \
  --data '{"source": "sitemap", "batch_size": 100}'
```

**What to Pre-Cache for Demo**:
1. Top 100 popular movies/shows from TMDB
2. All demo-specific titles (hackathon theme)
3. Search index for common queries ("action", "comedy", "2024")
4. Embeddings for semantic search demo queries

## Rate Limit Management

### Multi-Source Coordination

**TMDB Rate Limits** (Given):
- 50 requests/second per API key
- Burst capacity: Unknown (assume minimal)
- Reset: Per-second window

**JustWatch Rate Limits** (To Be Determined):
- Assume conservative: 10 requests/second
- May require backoff/retry logic

### Token Bucket Implementation

Based on 2025 best practices[¹³](#sources), token bucket allows burst handling while maintaining average rate:

```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;      // Max tokens (e.g., 50)
    this.tokens = capacity;        // Current tokens
    this.refillRate = refillRate;  // Tokens per second (e.g., 50)
    this.lastRefill = Date.now();
  }

  async consume(tokensNeeded = 1, priority = 'normal') {
    this.refill();

    if (this.tokens >= tokensNeeded) {
      this.tokens -= tokensNeeded;
      return true;
    }

    // Priority queue: user requests > background jobs
    if (priority === 'high') {
      await this.waitForTokens(tokensNeeded);
      this.tokens -= tokensNeeded;
      return true;
    }

    return false; // Reject or queue low-priority requests
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Usage
const tmdbBucket = new TokenBucket(50, 50);  // 50 req/sec
const justwatchBucket = new TokenBucket(10, 10);  // 10 req/sec (assumed)
```

### Priority Queuing Strategy

**Traffic Prioritization**[¹⁴](#sources):

| Request Type | Priority | Token Allocation | Behavior |
|--------------|----------|------------------|----------|
| **User API Requests** | High | Guaranteed (block if needed) | Wait up to 2s for tokens |
| **Real-time Availability** | High | Guaranteed | Wait up to 1s for tokens |
| **Background Enrichment** | Medium | Best-effort | Skip if no tokens available |
| **Bulk Metadata Sync** | Low | Throttled (max 20% capacity) | Queue for off-peak hours |
| **Demo Pre-caching** | Low | Off-peak only | Run during 2-6 AM UTC |

### Backoff Strategies

**Exponential Backoff with Jitter**[¹⁵](#sources):

```javascript
async function fetchWithBackoff(url, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await fetch(url);

      if (response.status === 429) {  // Rate limited
        const retryAfter = response.headers.get('Retry-After') ||
                           Math.pow(2, attempt);  // Exponential: 1s, 2s, 4s
        const jitter = Math.random() * 1000;  // 0-1s random jitter

        await sleep((retryAfter * 1000) + jitter);
        continue;
      }

      return response;
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
    }
  }
}
```

**Jitter Benefits**: Prevents "thundering herd" problem when multiple clients retry simultaneously[¹⁶](#sources).

### Rate Limit Headers (Client Communication)

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 50
X-RateLimit-Remaining: 23
X-RateLimit-Reset: 1701878400
Retry-After: 5
```

## Demo Performance Optimization

### Pre-Caching Strategy

**Phase 1: Core Data (Run 24h before demo)**
```bash
# 1. Popular titles from TMDB
- Top 100 movies (this year)
- Top 100 TV shows (this year)
- Top 50 trending (this week)

# 2. Hackathon theme-specific
- All "AI" or "Technology" themed content
- Award-winning films (Oscars, Emmys)
- High-rated documentaries

# 3. Streaming availability
- Netflix, Amazon Prime, Disney+ (US region)
- Pre-fetch for all cached titles

# Total: ~300-400 titles × 5KB = ~2 MB metadata
#        ~300-400 embeddings × 3KB = ~1.2 MB vectors
```

**Phase 2: Search Index Warming (Run 2h before demo)**
```bash
# Pre-compute vector searches for common queries
queries = [
  "action movies with AI",
  "best sci-fi series 2024",
  "documentaries about technology",
  "family-friendly adventure films",
  "thriller shows streaming now"
]

# Store top 50 results for each query
# Total: ~5 queries × 50 results × 5KB = ~1.25 MB
```

**Phase 3: Recommendation Precomputation (Run 1h before demo)**
```bash
# Generate recommendations for demo user profiles
profiles = ["action_fan", "family_viewer", "documentary_enthusiast"]

# Store top 20 recommendations per profile
# Total: ~3 profiles × 20 results × 5KB = ~300 KB
```

### Fallback Response Strategy

**Graceful Degradation Tiers**:

1. **Full Response** (Normal): Live API + cache + vector search
2. **Cached Response** (API slow): Serve cached data with staleness warning
3. **Static Fallback** (API down): Pre-loaded demo dataset (300-400 titles)
4. **Minimal Response** (Total failure): Return 503 with retry-after header

```javascript
async function fetchWithFallback(titleId) {
  try {
    // Tier 1: Try full response (timeout: 300ms)
    return await Promise.race([
      fetchFromAPI(titleId),
      sleep(300).then(() => { throw new Error('timeout'); })
    ]);
  } catch (error) {
    // Tier 2: Try cache (timeout: 100ms)
    const cached = await redis.get(`title:${titleId}`);
    if (cached) return { ...cached, _stale: true };

    // Tier 3: Try static fallback
    if (DEMO_DATASET[titleId]) {
      return { ...DEMO_DATASET[titleId], _fallback: true };
    }

    // Tier 4: Return error
    throw new Error('Service temporarily unavailable');
  }
}
```

### Metrics to Show Judges

**Performance Dashboard** (Real-time during demo):

```plaintext
┌─────────────────────────────────────────────────────────┐
│  TV5 Media Gateway - Live Performance Metrics          │
├─────────────────────────────────────────────────────────┤
│  API Response Time:                                     │
│    P50: 45ms  ████████░░  P95: 120ms  P99: 250ms      │
│                                                         │
│  Cache Hit Rate:                                        │
│    Metadata: 94% ████████████░  Vector: 78% ███████░   │
│                                                         │
│  Rate Limiting:                                         │
│    TMDB: 23/50 req/s  ████░░░░░░  JustWatch: 4/10      │
│                                                         │
│  Vector Search:                                         │
│    QPS: 145  Latency: 85ms  Recall: 95%                │
│                                                         │
│  Database:                                              │
│    Queries: 12/sec  Connections: 8/100  Pool: 8%       │
└─────────────────────────────────────────────────────────┘
```

**Key Metrics to Highlight**:
1. **Sub-200ms p95 latency** (industry-leading for aggregated metadata)
2. **95%+ cache hit rate** (efficient resource usage)
3. **Zero rate limit errors** (smart throttling)
4. **95%+ vector search recall** (semantic accuracy)
5. **10K+ titles searchable** (scale demonstration)

### Load Testing Approach

**Pre-Demo Validation**:

```bash
# Tool: Apache Bench or k6
k6 run --vus 50 --duration 5m load-test.js

# Scenarios
1. Concurrent browsing (50 users, random titles)
2. Search storm (20 users, same query simultaneously)
3. Recommendation generation (10 users, personalized)
4. Mixed traffic (50 users, realistic distribution)

# Success Criteria
- P95 latency < 300ms for all scenarios
- Zero 5xx errors
- Cache hit rate > 85%
- No rate limit errors (429)
```

## Monitoring Strategy

### Key Metrics to Track

**Application-Level**[¹⁷](#sources):

| Metric | Collection Method | Alert Threshold | Priority |
|--------|-------------------|-----------------|----------|
| **API Latency (p50/p95/p99)** | StatsD/Prometheus | P95 > 300ms for 5min | Critical |
| **Error Rate** | Application logs | >1% errors for 2min | Critical |
| **Cache Hit Rate** | Redis INFO | <80% for 10min | Warning |
| **Rate Limit Usage** | Token bucket counters | >90% capacity | Warning |
| **Vector Search QPS** | Database metrics | <50 QPS (degradation) | Warning |

**Infrastructure-Level**:

| Metric | Tool | Alert Threshold | Priority |
|--------|------|-----------------|----------|
| **CPU Usage** | cAdvisor/Prometheus | >80% for 5min | Warning |
| **Memory Usage** | cAdvisor/Prometheus | >90% available | Critical |
| **Disk I/O** | Node Exporter | >80% utilization | Warning |
| **Network Latency** | Ping/Traceroute | >50ms to external APIs | Warning |

### Alerting Strategy

**Tiered Severity**[¹⁸](#sources):

```yaml
Critical Alerts (PagerDuty):
  - P95 latency > 500ms for 2 minutes
  - Error rate > 5% for 1 minute
  - Service down (no heartbeat)
  - Memory usage > 95%

Warning Alerts (Slack):
  - P95 latency > 300ms for 5 minutes
  - Cache hit rate < 80% for 10 minutes
  - Rate limit usage > 90% for 5 minutes
  - CPU usage > 80% for 10 minutes

Info Alerts (Dashboard):
  - Unusual traffic patterns
  - Slow external API responses
  - Cache evictions increase
```

### RED Method Implementation

**Rate, Errors, Duration** (microservices best practice)[¹⁹](#sources):

```javascript
// Prometheus metrics
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3, 5]  // Percentile calc
});

const httpRequestErrors = new Counter({
  name: 'http_request_errors_total',
  help: 'Total HTTP errors',
  labelNames: ['method', 'route', 'error_type']
});
```

### Dashboard Design

**Real-Time Performance View** (Grafana/DataDog):

```plaintext
Row 1: API Performance
  - Request Rate (requests/sec, 5min avg)
  - Latency Percentiles (p50/p95/p99, line chart)
  - Error Rate (%, 5min rolling window)

Row 2: Caching
  - Cache Hit Rate by Layer (L1/L2/L3, stacked bar)
  - Cache Memory Usage (%, gauge)
  - Top Cached Items (table, by hit count)

Row 3: External APIs
  - TMDB Rate Limit Usage (%, gauge)
  - JustWatch Rate Limit Usage (%, gauge)
  - External API Latency (ms, line chart)

Row 4: Vector Search
  - QPS (queries/sec, line chart)
  - Search Latency (ms, p95, line chart)
  - Recall @ K (%, gauge)

Row 5: Infrastructure
  - CPU/Memory Usage (%, dual gauge)
  - Database Connections (count, gauge)
  - Network I/O (MB/s, line chart)
```

## Recommendations

### Immediate (Hackathon MVP)

| Aspect | Recommendation | Rationale | Impact |
|--------|----------------|-----------|--------|
| **Caching** | Implement 2-tier caching (in-memory + Redis) with 24h TTL for metadata | 95%+ cache hit rate achievable, reduces API calls by 80-95%[²⁰](#sources) | High |
| **Rate Limiting** | Token bucket with priority queue (user > background) | Prevents API overages, ensures user requests succeed | Critical |
| **Vector DB** | Use Qdrant or Milvus with HNSW index for 100K vectors | Proven 1,200+ QPS at 95% recall[²¹](#sources) | High |
| **Pre-Caching** | Warm cache with 300-400 popular titles 24h before demo | Eliminates cold start latency, ensures smooth demo | Critical |
| **Monitoring** | Basic Prometheus + Grafana with p95 latency alerts | Early detection of issues, data-driven optimization | Medium |

### Production Scale (1M+ Titles)

| Aspect | Recommendation | Rationale | Impact |
|--------|----------------|-----------|--------|
| **Caching** | Redis Cluster (3+ nodes) with binary quantization for embeddings | Reduces memory by 40x (550 MB vs 22 GB)[²²](#sources) | Critical |
| **Rate Limiting** | Distributed token bucket with Redis (shared state) | Horizontal scaling, consistent rate limiting across nodes | High |
| **Vector DB** | Sharded Milvus/Qdrant with GPU acceleration | Handles 10M+ vectors at 2,000+ QPS[²³](#sources) | High |
| **Database** | PostgreSQL read replicas (3+) with connection pooling | Distributes read load, prevents connection exhaustion | High |
| **CDN** | CloudFlare/Fastly for static metadata + edge caching | Reduces origin load, <50ms latency globally | Medium |
| **Observability** | Full distributed tracing (Jaeger/Tempo) + APM | Root cause analysis, performance bottleneck identification | Medium |

### Cost Optimization

| Strategy | Savings | Trade-off |
|----------|---------|-----------|
| **Binary Quantization** | 40x memory reduction[²⁴](#sources) | 2-5% recall degradation (acceptable for most use cases) |
| **TTL-based Invalidation** | Eliminates 70% event processing overhead | Maximum 24h staleness (acceptable for metadata) |
| **Off-peak Batch Jobs** | 50% cheaper compute during 2-6 AM | Slight delay in background enrichment |
| **Serverless Functions** | Pay-per-use for infrequent tasks | Cold start latency (200-500ms) |

### Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **TMDB Rate Limit Hit** | High (without throttling) | Critical (service down) | Token bucket + priority queue + cache warming |
| **Cache Stampede** | Medium (on popular content) | High (database overload) | TTL jitter + stale-while-revalidate pattern |
| **Vector Search Slow** | Medium (with scale) | Medium (poor UX) | HNSW index + binary quantization + GPU acceleration |
| **External API Down** | Low (TMDB/JustWatch SLA) | High (no fresh data) | Static fallback dataset + long TTL cache |
| **Memory Exhaustion** | Medium (without limits) | Critical (OOM crashes) | LRU eviction policy + memory limits + alerts |

## Technical Architecture Diagram

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                         User Request                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │
                    │ (Rate Limiting) │
                    │ Token Bucket:   │
                    │ - TMDB: 50/s    │
                    │ - JustWatch:10/s│
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
       ┌──────▼──────┐             ┌───────▼────────┐
       │  L1 Cache   │             │   L2 Cache     │
       │ (In-Memory) │             │    (Redis)     │
       │             │             │                │
       │ • 100-500MB │             │ • 1.5-15 GB    │
       │ • <5ms      │             │ • 10-30ms      │
       │ • TTL: 5min │    miss     │ • TTL: 1-24h   │
       └──────┬──────┘    ────────>└───────┬────────┘
              │ hit                         │ miss
              │                             │
              └──────────────┬──────────────┘
                             │ both miss
                             │
                    ┌────────▼────────┐
                    │  Data Sources   │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │ PostgreSQL  │ │ ← Metadata storage
                    │ └─────────────┘ │
                    │ ┌─────────────┐ │
                    │ │ Qdrant/     │ │ ← Vector embeddings
                    │ │ Milvus      │ │   (HNSW index)
                    │ └─────────────┘ │
                    │ ┌─────────────┐ │
                    │ │ TMDB API    │ │ ← External metadata
                    │ └─────────────┘ │
                    │ ┌─────────────┐ │
                    │ │ JustWatch   │ │ ← Streaming availability
                    │ └─────────────┘ │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │   Monitoring    │
                    │                 │
                    │ • Prometheus    │
                    │ • Grafana       │
                    │ • Alerts        │
                    └─────────────────┘
```

## Performance Testing Checklist

**Pre-Demo Validation**:

- [ ] Load test: 50 concurrent users for 5 minutes
- [ ] Cache hit rate: >90% after warm-up
- [ ] P95 latency: <300ms for all endpoints
- [ ] Rate limiting: Zero 429 errors under load
- [ ] Vector search: >95% recall, <150ms p95 latency
- [ ] Failover: Test external API timeout handling
- [ ] Memory: Stable under sustained load (no leaks)
- [ ] Dashboard: All metrics visible and accurate

**Demo Day Checklist**:

- [ ] Cache warmed 24h prior (300-400 titles)
- [ ] Search index precomputed (top 10 queries)
- [ ] Monitoring dashboard running (projector-ready)
- [ ] Fallback dataset loaded (static 100 titles)
- [ ] Alert thresholds raised (demo mode)
- [ ] Performance metrics logged (for post-demo analysis)

---

## Sources

1. [Milvus - Storage Requirements for Embeddings](https://milvus.io/ai-quick-reference/what-are-the-storage-requirements-for-embeddings) - Vector storage calculation
2. [Qdrant - Binary Quantization](https://qdrant.tech/articles/binary-quantization/) - Memory reduction techniques
3. [Scaling Vector Databases: RAM Usage](https://stevescargall.com/blog/2024/08/how-much-ram-could-a-vector-database-use-if-a-vector-database-could-use-ram/) - Memory requirements for vector DBs
4. [Arcjet - 25ms p95 Response Time SLA](https://blog.arcjet.com/how-we-achieve-our-25ms-p95-response-time-sla/) - Real-world latency targets
5. [OneUpTime - P50 vs P95 vs P99 Latency](https://oneuptime.com/blog/post/2025-09-15-p50-vs-p95-vs-p99-latency-percentiles/view) - Percentile metrics explained
6. [Medium - Understanding P95/P99 Latency Principle](https://belowthemalt.com/2025/09/01/understanding-the-p95-p99-latency-principle-why-the-slowest-requests-matter-most/) - Why tail latency matters
7. [Aerospike - What Is P99 Latency?](https://aerospike.com/blog/what-is-p99-latency/) - P99 importance in production
8. [Odown - API Response Time Standards](https://odown.com/blog/api-response-time-standards/) - Industry benchmarks
9. [AWS - Cache Validity Strategies](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/cache-validity.html) - TTL best practices
10. [Terabyte Systems - Redis Caching Strategies](https://terabyte.systems/posts/redis-caching-strategies-best-practices/) - TTL jitter pattern
11. [Daily.dev - Cache Invalidation vs Expiration](https://daily.dev/blog/cache-invalidation-vs-expiration-best-practices) - Hybrid invalidation approaches
12. [Ioriver - Cache Warming](https://www.ioriver.io/terms/cache-warming) - Pre-caching strategies
13. [Wikipedia - Token Bucket](https://en.wikipedia.org/wiki/Token_bucket) - Token bucket algorithm
14. [Eraser - Token Bucket vs Leaky Bucket](https://www.eraser.io/decision-node/api-rate-limiting-strategies-token-bucket-vs-leaky-bucket) - Priority queue integration
15. [Zuplo - 10 Best Practices for API Rate Limiting](https://zuplo.com/learning-center/10-best-practices-for-api-rate-limiting-in-2025) - Exponential backoff
16. [Kodekx - API Rate Limiting Best Practices](https://www.kodekx.com/blog/api-rate-limiting-best-practices-scaling-saas-2025) - Jitter for thundering herd
17. [SigNoz - API Monitoring Complete Guide](https://signoz.io/blog/api-monitoring-complete-guide/) - Monitoring metrics
18. [Catchpoint - API Performance Monitoring](https://www.catchpoint.com/api-monitoring-tools/api-performance-monitoring) - Alert thresholds
19. [API7 - Rate Limiting Strategies](https://api7.ai/learning-center/api-101/rate-limiting-strategies-for-api-management) - RED method
20. [Redis - Why Caching Strategies Hold You Back](https://redis.io/blog/why-your-caching-strategies-might-be-holding-you-back-and-what-to-consider-next/) - Cache hit rate benefits
21. [Qdrant - Vector Database Benchmarks](https://qdrant.tech/benchmarks/) - QPS and recall metrics
22. [Pure Storage - Managing Vector Storage Bloat](https://blog.purestorage.com/purely-technical/managing-vector-storage-bloat-insights-for-scalable-systems/) - Quantization savings
23. [TopK Bench - Benchmarking Real-World Vector Search](https://www.topk.io/blog/20251201-topk-bench) - Large-scale performance
24. [Firecrawl - Best Vector Databases 2025](https://www.firecrawl.dev/blog/best-vector-databases-2025) - Vector DB comparison
25. [Tinybird - Distributed Caching for Real-Time Systems](https://www.tinybird.co/blog-posts/distributed-caching-for-scalable-real-time-systems-sb) - Distributed caching patterns
26. [Netflix Tech Blog - Real-Time Distributed Graph](https://netflixtechblog.com/how-and-why-netflix-built-a-real-time-distributed-graph-part-1-ingesting-and-processing-data-80113e124acc) - Real-time metadata architecture
27. [CockroachDB - Database Hotspots in Media Streaming](https://www.cockroachlabs.com/blog/the-hot-content-problem-metadata-storage-for-media-streaming/) - Media metadata challenges

---

**Research completed by**: B6-PERF Agent
**Phase**: A - Discovery & Evaluation
**Next steps**: Review with architecture team, integrate with API evaluation findings, validate with load testing
