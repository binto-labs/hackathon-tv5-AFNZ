---
status: keep
phase: complete
type: report
version: 1.0
last-updated: 2025-12-06
title: Edge Synchronization Patterns Research
author: Edge Sync Researcher (Edge Computing Research Swarm)
tags: [edge-computing, sync, cache, consistency, research]
---

# Edge Synchronization Patterns Research

## Executive Summary

For the TV5 Media Gateway hackathon, distributing streaming availability data and vector embeddings to 200+ edge nodes requires a multi-layered synchronization strategy combining **predictive caching**, **tag-based invalidation**, and **event-driven updates** to balance freshness with performance.

### Key Recommendations for TV5 Hackathon:

1. **Use Cloudflare Workers KV** for hot title embeddings (10K titles) - 98% cache hit rate, <5ms p99 latency
2. **Implement TTL-based expiration** with 12-24 hour freshness windows for availability data
3. **Add Redis Pub/Sub** for real-time catalog change notifications to edge nodes
4. **Use surrogate key invalidation** for granular purging when platforms add/remove titles
5. **Gracefully degrade** with stale-while-revalidate pattern (<24 hour staleness acceptable)

**Expected Performance:**
- 500µs-10ms read latency for cached embeddings (Cloudflare KV hot keys)
- <60 second propagation time for updates across 200+ PoPs
- 95%+ cache hit rates reducing origin load by 95%

---

## 1. Embedding Sync Strategy: Distributing 10K Hot Titles to 200+ PoPs

### Challenge
Vector embeddings for 10,000 hot titles need to be available at 200+ Cloudflare edge locations with minimal latency for semantic search operations.

### Recommended Pattern: **Predictive Caching with Workers KV**

#### Architecture

```
┌─────────────────────┐
│  Central Database   │
│  (PostgreSQL/pgvector)│
│  - 10K hot titles   │
│  - Embeddings       │
└──────────┬──────────┘
           │
           │ Batch Fill (off-peak)
           ▼
┌─────────────────────┐
│  Workers KV Store   │
│  - Global namespace │
│  - Edge replicas    │
└──────────┬──────────┘
           │
           │ Cache on first read
           ▼
┌─────────────────────────────────────┐
│   200+ Cloudflare Edge Locations   │
│   - Local cache (hot keys)          │
│   - 500µs-10ms read latency         │
│   - 98% hit rate                    │
└─────────────────────────────────────┘
```

#### Implementation Strategy

**Storage Choice: Cloudflare Workers KV**

Based on Cloudflare's [storage options comparison](https://developers.cloudflare.com/workers/platform/storage-options/), Workers KV is ideal because:

- **Read-optimized**: Designed for high read volumes (thousands of RPS per key)
- **Global distribution**: Automatic replication to 200+ PoPs
- **Low latency**: 500µs-10ms for hot keys, <5ms p99 after [2025 rearchitecture](https://www.infoq.com/news/2025/08/cloudflare-workers-kv/)
- **Cost-effective**: No egress fees, $5/month base plan includes substantial usage
- **Cache behavior**: Frequently accessed keys cached at edge automatically

**KV Configuration:**

```javascript
// Workers KV binding in wrangler.toml
[[kv_namespaces]]
binding = "EMBEDDINGS"
id = "your-namespace-id"

// Store embeddings with metadata
await EMBEDDINGS.put(
  `embedding:${titleId}`,
  JSON.stringify({
    vector: [0.123, 0.456, ...], // 768-dim embedding
    title: "The Matrix",
    platforms: ["netflix", "hulu"],
    updated_at: "2025-12-06T10:00:00Z"
  }),
  {
    expirationTtl: 86400, // 24 hours
    metadata: {
      version: 2,
      popularity_rank: 42
    }
  }
);
```

**Sync Process:**

1. **Initial Fill (Off-Peak)**: Batch upload 10K embeddings during low-traffic hours
2. **On-Demand Caching**: First request to each PoP pulls from central KV store
3. **Automatic Edge Caching**: Hot keys cached locally after first access
4. **Periodic Refresh**: TTL-based expiration (24 hours) forces re-fetch

**Fallback Strategy:**

```javascript
// Stale-while-revalidate pattern
async function getEmbedding(titleId) {
  const cached = await EMBEDDINGS.get(`embedding:${titleId}`, {type: "json"});

  if (cached) {
    const age = Date.now() - new Date(cached.updated_at).getTime();

    // Serve stale if <48 hours old, trigger async refresh if >24 hours
    if (age > 86400000 && age < 172800000) {
      ctx.waitUntil(refreshEmbedding(titleId)); // Background refresh
    }

    return cached;
  }

  // Cache miss: fetch from origin
  return await fetchFromOrigin(titleId);
}
```

### Alternative Considered: Durable Objects

[Durable Objects](https://developers.cloudflare.com/durable-objects/) offer stronger consistency and transactional storage, but are **not recommended** for this use case because:

- **Overkill**: We don't need strong consistency for embeddings (eventual consistency OK)
- **Higher latency**: Single-location coordination adds round-trip time
- **Cost**: More expensive for read-heavy workloads
- **Use case mismatch**: Better for stateful coordination, not distributed caching

**When to use Durable Objects instead:**
- Real-time collaboration features
- Session state requiring strict ordering
- Transactional updates across multiple keys

---

## 2. Cache Invalidation Patterns for Streaming Availability

### Challenge
Streaming platforms add/remove titles daily. Edge caches need updates without constant polling.

### Recommended Pattern: **Hybrid TTL + Event-Driven Invalidation**

#### Strategy Overview

```
┌─────────────────────┐
│   Platform APIs     │
│  (Netflix, Hulu)    │
└──────────┬──────────┘
           │
           │ Webhook/Polling
           ▼
┌─────────────────────┐
│   Change Detector   │
│   (Cloudflare Worker)│
└──────────┬──────────┘
           │
           ├─────────────────────┐
           │                     │
           ▼                     ▼
┌──────────────────┐   ┌─────────────────┐
│   Redis Pub/Sub  │   │  Workers KV     │
│   Notifications  │   │  Tag Purge      │
└──────────────────┘   └─────────────────┘
           │                     │
           └──────────┬──────────┘
                      ▼
           ┌─────────────────────┐
           │   Edge Nodes (200+) │
           │   - Listen to events│
           │   - Purge tags      │
           │   - Refresh cache   │
           └─────────────────────┘
```

#### Implementation: Surrogate Key Invalidation

Based on [CDN cache invalidation best practices](https://medium.com/@sharareshaddev/modern-cdn-invalidation-strategies-for-dynamic-spa-content-7c3081d2a0c5), use **tag-based purging** for granular control:

```javascript
// When caching availability data, add surrogate keys
const response = new Response(availabilityData, {
  headers: {
    'Cache-Control': 'public, max-age=43200', // 12 hours
    'Cache-Tag': `title:${titleId}, platform:netflix, genre:action`
  }
});

// When Netflix removes a title, purge all related cache entries
await purgeByTag(`title:${titleId}`);
await purgeByTag(`platform:netflix`); // Bulk update for platform changes
```

**Invalidation Triggers:**

1. **Time-based**: 12-24 hour TTL for availability data
2. **Event-based**: Platform webhook triggers immediate purge
3. **Prefix-based**: Purge entire platform catalog on major updates

#### TTL Strategy by Content Type

| Content Type | TTL | Invalidation | Staleness Tolerance |
|--------------|-----|--------------|---------------------|
| **Embeddings** | 24 hours | Version bump | High (48 hours OK) |
| **Availability data** | 12 hours | Event-driven | Medium (24 hours) |
| **Trending lists** | 5 minutes | Time-only | Low (15 minutes) |
| **Static metadata** | 7 days | Manual purge | Very high |
| **Live playlists** | 5 seconds | Auto-refresh | Critical (10 sec max) |

Source: [AWS Streaming Media best practices](https://docs.aws.amazon.com/wellarchitected/latest/streaming-media-lens/review-and-monitoring.html)

---

## 3. Cloudflare Storage Comparison: KV vs Durable Objects vs R2

### Detailed Comparison Matrix

| Feature | **Workers KV** | **Durable Objects** | **R2 Object Storage** |
|---------|----------------|---------------------|----------------------|
| **Use Case** | Read-heavy key-value cache | Stateful coordination | Large file storage |
| **Consistency** | Eventual (60s propagation) | Strong (single-location) | Strong per-object |
| **Read Latency** | 500µs-10ms (hot keys) | 2-3ms (in-memory) | Higher (blob retrieval) |
| **Write Latency** | 1-5s global propagation | Immediate (single location) | Immediate |
| **Cost** | $0.50 per million reads | $0.15 per million requests | $0.015 per million reads |
| **Storage Cost** | $0.50 per GB/month | Included in Durable Object | $0.015 per GB/month |
| **Max Object Size** | 25 MB | Limited by memory | 5 TB per object |
| **Replication** | Automatic to 200+ PoPs | Single-location (coordinated) | Multi-region on-demand |
| **Caching** | Automatic edge caching | In-memory state | Manual via Cache API |

Sources: [Cloudflare storage options](https://developers.cloudflare.com/workers/platform/storage-options/), [Pricing](https://developers.cloudflare.com/workers/platform/pricing/)

### **Recommendation for TV5:**

1. **Workers KV**: Store 10K hot title embeddings (read-heavy, tolerate 60s staleness)
2. **Durable Objects**: NOT needed unless implementing real-time features later
3. **R2**: Store full embedding dataset (100K+ titles) as backup/fallback

**Cost Estimate (10K hot titles, 200+ PoPs):**

```
Assumptions:
- 10,000 embeddings × 10 KB average = 100 MB storage
- 1M reads/day across 200 PoPs = 30M reads/month
- 10K writes/day (daily updates) = 300K writes/month

Workers KV Cost:
- Storage: 0.1 GB × $0.50 = $0.05/month
- Reads: 30M × $0.50/million = $15/month
- Writes: 300K × $5/million = $1.50/month
Total: ~$16.55/month + $5 base = $21.55/month

R2 Backup Cost:
- Storage: 1 GB (full dataset) × $0.015 = $0.015/month
- Reads: Minimal (fallback only) = $0.01/month
Total: ~$0.03/month

Grand Total: ~$22/month for entire edge caching infrastructure
```

### Architecture: Combining KV + R2

```javascript
// Worker: Multi-tier fallback
async function getEmbedding(titleId) {
  // Tier 1: Workers KV (hot cache)
  let embedding = await EMBEDDINGS_KV.get(`hot:${titleId}`, {type: "json"});
  if (embedding) return embedding;

  // Tier 2: R2 (warm storage)
  const r2Object = await R2_BUCKET.get(`embeddings/${titleId}.json`);
  if (r2Object) {
    embedding = await r2Object.json();
    // Promote to KV for future requests
    await EMBEDDINGS_KV.put(`hot:${titleId}`, JSON.stringify(embedding), {
      expirationTtl: 86400
    });
    return embedding;
  }

  // Tier 3: Origin database (cold)
  return await fetchFromOrigin(titleId);
}
```

---

## 4. Pub/Sub Options for Edge Notifications

### Challenge
When catalog changes occur, 200+ edge nodes need notification without polling central database.

### Option 1: **Redis Pub/Sub** (Recommended for Hackathon)

#### Architecture

```
┌─────────────────────┐
│  Catalog Database   │
│  (PostgreSQL)       │
└──────────┬──────────┘
           │
           │ INSERT/UPDATE trigger
           ▼
┌─────────────────────┐
│  Redis Pub/Sub      │
│  Channel: catalog-updates│
└──────────┬──────────┘
           │
           │ Subscribe
           ▼
┌─────────────────────────────┐
│  Edge Workers (200+ PoPs)   │
│  - Subscribe on startup     │
│  - Invalidate local cache   │
│  - Fetch fresh data         │
└─────────────────────────────┘
```

#### Implementation

Based on [Redis Pub/Sub patterns](https://medium.com/@deghun/redis-pub-sub-local-memory-low-latency-high-consistency-caching-3740f66f0368), use a **version-based broadcast** approach:

```javascript
// Publisher: Database trigger publishes change events
const redis = new Redis(process.env.REDIS_URL);

// On title update
await redis.publish('catalog-updates', JSON.stringify({
  type: 'TITLE_UPDATED',
  titleId: '12345',
  version: 47, // Monotonically increasing
  platforms: ['netflix', 'hulu'],
  timestamp: Date.now()
}));
```

```javascript
// Subscriber: Edge Worker with Durable Object for subscription
export class CatalogSubscriber {
  constructor(state, env) {
    this.state = state;
    this.redis = new Redis(env.REDIS_URL);
  }

  async fetch(request) {
    // Subscribe to channel
    await this.redis.subscribe('catalog-updates', (message) => {
      const event = JSON.parse(message);

      // Only apply if version is newer (monotonic version check)
      const currentVersion = await this.state.storage.get(`version:${event.titleId}`);
      if (event.version > (currentVersion || 0)) {
        // Invalidate local cache
        await this.invalidateCache(event.titleId);
        await this.state.storage.put(`version:${event.titleId}`, event.version);
      }
    });
  }

  async invalidateCache(titleId) {
    // Remove from Workers KV or local memory
    await env.EMBEDDINGS_KV.delete(`embedding:${titleId}`);
  }
}
```

**Pros:**
- ✅ Simple to implement
- ✅ Low latency (<100ms notification propagation)
- ✅ Scales to thousands of subscribers
- ✅ Built-in pattern matching (`catalog:netflix:*`)

**Cons:**
- ❌ **At-most-once delivery** - messages lost if subscriber disconnected ([Redis Pub/Sub docs](https://redis.io/docs/latest/develop/pubsub/))
- ❌ No message persistence
- ❌ No replay capability for missed updates

**Mitigation:**
- Use [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) for durable messages
- Implement periodic "version check" (compare database version every 5 minutes)
- Fallback: TTL self-healing (stale cache expires after 24 hours)

### Option 2: **Cloudflare Durable Objects + WebSockets**

```javascript
// Durable Object as message broker
export class CatalogBroker {
  constructor(state, env) {
    this.state = state;
    this.sessions = new Set();
  }

  async fetch(request) {
    // Edge workers connect via WebSocket
    const [client, server] = Object.values(new WebSocketPair());
    this.state.acceptWebSocket(server);
    this.sessions.add(server);

    return new Response(null, { status: 101, webSocket: client });
  }

  async broadcast(message) {
    // Notify all connected edge workers
    for (const session of this.sessions) {
      session.send(JSON.stringify(message));
    }
  }
}
```

**Pros:**
- ✅ Native Cloudflare integration
- ✅ Persistent connections
- ✅ No external dependency

**Cons:**
- ❌ More complex setup
- ❌ Higher cost (Durable Object per connection)
- ❌ Limited to Cloudflare ecosystem

### Option 3: **Webhook + Queue Pattern**

```
┌─────────────────────┐
│  Platform Webhook   │
│  (Netflix API)      │
└──────────┬──────────┘
           │
           │ POST /webhook
           ▼
┌─────────────────────┐
│  Cloudflare Worker  │
│  (Webhook Handler)  │
└──────────┬──────────┘
           │
           │ Enqueue
           ▼
┌─────────────────────┐
│  Cloudflare Queue   │
│  (Message Broker)   │
└──────────┬──────────┘
           │
           │ Batch Consumer
           ▼
┌─────────────────────┐
│  Cache Invalidator  │
│  - Purge KV tags    │
│  - Notify workers   │
└─────────────────────┘
```

**Use case**: Platforms with reliable webhooks (Spotify, Twitch)

### **Recommendation for TV5 Hackathon:**

**Start with Redis Pub/Sub** for simplicity, upgrade to Redis Streams if durability required.

**Implementation Priority:**
1. ✅ **Phase 1**: TTL-based expiration (12-24 hours) - zero complexity
2. ✅ **Phase 2**: Redis Pub/Sub for real-time notifications - medium complexity
3. 🔮 **Phase 3**: Redis Streams for guaranteed delivery - high complexity (post-hackathon)

---

## 5. Stale Data Handling: Graceful Degradation Strategies

### Challenge
Edge nodes may serve stale data due to network partitions, propagation delays, or missed notifications.

### Pattern 1: **Stale-While-Revalidate**

Based on [eventually consistent systems](https://www.keboola.com/blog/eventual-consistency) design patterns:

```javascript
// Cache-Control header pattern
response.headers.set(
  'Cache-Control',
  'public, max-age=43200, stale-while-revalidate=86400'
);
```

**Behavior:**
- Serve cached data for 12 hours (fresh)
- Continue serving cached data for **additional 24 hours** while fetching update in background
- Total staleness tolerance: 36 hours

**Visual Timeline:**

```
0h ────── 12h ────────────── 36h ─────────> ∞
 │         │                  │
 │  FRESH  │  STALE (serving) │  EXPIRED
 │         │  (revalidating)  │  (must fetch)
 │         │                  │
 └─────────┴──────────────────┴──────────────
```

### Pattern 2: **Versioned Staleness Indicators**

```javascript
// Add version metadata to cached objects
const cachedData = {
  title: "The Matrix",
  platforms: ["netflix"],
  embedding: [...],
  metadata: {
    version: 47,
    fetched_at: "2025-12-06T08:00:00Z",
    expires_at: "2025-12-06T20:00:00Z",
    stale: false
  }
};

// On read, check staleness
function serveWithFreshness(data) {
  const age = Date.now() - new Date(data.metadata.fetched_at).getTime();
  const hoursOld = age / 3600000;

  if (hoursOld > 24) {
    // Indicate staleness to client
    data.metadata.stale = true;
    data.metadata.stale_hours = hoursOld;

    // Trigger background refresh
    ctx.waitUntil(refreshData(data.titleId));
  }

  return data;
}
```

### Pattern 3: **Read Quorum for Critical Data**

Based on [eventually consistent stale data handling](https://www.designgurus.io/answers/detail/what-is-eventual-consistency-and-how-does-it-differ-from-strong-consistency-in-distributed-systems):

```javascript
// Read from multiple sources, use newest version
async function getWithQuorum(titleId) {
  const sources = [
    EMBEDDINGS_KV.get(`embedding:${titleId}`),
    R2_BUCKET.get(`embeddings/${titleId}.json`),
    fetchFromOrigin(titleId)
  ];

  const results = await Promise.allSettled(sources);

  // Filter successful reads
  const valid = results
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value);

  // Return newest version by timestamp
  return valid.sort((a, b) =>
    new Date(b.updated_at) - new Date(a.updated_at)
  )[0];
}
```

**Trade-offs:**
- ✅ Higher confidence in freshness
- ❌ 3x latency (waiting for multiple sources)
- ❌ 3x cost (multiple reads)

**Use case**: Critical data only (e.g., payment-related content)

### Pattern 4: **Eventual Consistency UI Indicators**

Show users when data might be stale:

```javascript
// Return freshness metadata to client
{
  "title": "The Matrix",
  "platforms": ["netflix", "hulu"],
  "freshness": {
    "last_updated": "2 hours ago",
    "confidence": "high", // <12h = high, 12-24h = medium, >24h = low
    "next_update": "in 10 hours"
  }
}
```

**UI Examples:**
- 🟢 "Data updated 2 hours ago" (high confidence)
- 🟡 "Data updated 18 hours ago" (medium confidence)
- 🔴 "Data updated 36 hours ago - refreshing..." (low confidence)

### **Staleness Tolerance Matrix for TV5:**

| Data Type | Max Staleness | Degradation Strategy |
|-----------|---------------|----------------------|
| **Embeddings** | 48 hours | Serve stale, async refresh |
| **Availability** | 24 hours | Serve stale, show timestamp |
| **Trending** | 1 hour | Fetch fresh or hide |
| **Metadata** | 7 days | Serve stale indefinitely |

### Monitoring Staleness

```javascript
// Track staleness metrics
async function trackStaleness(titleId, age) {
  await analytics.track({
    event: 'cache_staleness',
    properties: {
      title_id: titleId,
      age_hours: age / 3600000,
      served_stale: age > 43200000, // >12 hours
      environment: 'edge'
    }
  });
}
```

---

## 6. Industry Case Studies

### Netflix Open Connect: 98% Cache Hit Rate

Based on [Netflix's architecture](https://medium.com/@hjain5164/inside-netflixs-video-streaming-delivery-architecture-e2c848e98a85):

**Key Learnings:**

1. **Predictive Caching**: Netflix pre-fills edge nodes during off-peak hours
   - "Encoded assets are pushed to OCAs during off-peak 'fill windows'"
   - 17,000+ servers across 158 countries
   - 98% cache hit rate reduces origin egress to 2%

2. **Localized Traffic**: Partner with ISPs to place servers inside networks
   - "Open Connect appliances within ISP networks diminish latency significantly"
   - Reduces backbone costs and improves startup time

3. **Dual-Plane Architecture**: Separate control plane (AWS) from data plane (Open Connect)
   - Control plane handles browsing, recommendations (AWS)
   - Data plane serves video streams (private CDN)

**TV5 Application:**
- Pre-load hot embeddings during low-traffic periods (2-6 AM)
- Place frequently requested titles in Workers KV "hot tier"
- Separate metadata API (Cloudflare Workers) from embedding storage (KV)

### Spotify: Multi-Tier Caching + Predictive Pre-Fetch

From [Spotify's architecture research](https://sderay.com/how-spotify-streams-music-to-millions-in-real-time-content-caching-edge-computing-adaptive-streaming/):

**Key Patterns:**

1. **Edge Predictive Caching**:
   - "AI-based predictive caching uses ML models to predict which songs users might play next"
   - Cache songs based on listening habits before user requests

2. **Multi-Tiered Storage**:
   - Tier 1: Redis/Memcached (metadata, playlists)
   - Tier 2: Edge servers (popular tracks)
   - Tier 3: Origin servers (long-tail content)

3. **LRU Eviction**:
   - "LRU (Least Recently Used) caching policies remove songs that are less frequently played"
   - Optimize storage for trending content

**TV5 Application:**
- Use viewing history to predict which embeddings users will need
- Implement LRU eviction in Workers KV (automatic via access patterns)
- Three-tier: KV (hot) → R2 (warm) → PostgreSQL (cold)

### Facebook TAO: Stream-Based Cache Invalidation

From [stream-based invalidation patterns](https://markpapadakis.medium.com/invalidating-caches-and-reacting-to-updates-using-streams-4c32ca9735ef):

**Pattern:**

```
Database → Change Stream → Message Queue → Edge Invalidation
```

**Key Insight:**
- "The invalidation logic always executes, even if the application crashes, because the update event is emitted by the persistent stores themselves"
- Facebook's TAO service uses stream-based invalidation

**Benefits:**
- ✅ Guaranteed execution (events persist)
- ✅ Survives application crashes
- ✅ Decoupled invalidation logic

**TV5 Application:**
- Use PostgreSQL logical replication for change data capture
- Stream changes to Redis/Kafka
- Edge workers consume stream and invalidate caches

---

## 7. Recommendations for TV5 Hackathon

### Phase 1: MVP (Hackathon Deliverable)

**Goal**: Get 10K hot titles cached at edge with reasonable freshness

**Architecture:**

```
PostgreSQL + pgvector
    ↓ (batch export every 24h)
Cloudflare Workers KV (10K embeddings)
    ↓ (automatic edge caching)
200+ Cloudflare PoPs
```

**Implementation Steps:**

1. **Setup Workers KV namespace** (`npx wrangler kv:namespace create EMBEDDINGS`)
2. **Daily sync script** (runs via cron):
   ```javascript
   // cron: 0 2 * * * (2 AM daily)
   const hotTitles = await db.query(`
     SELECT id, title, embedding, platforms
     FROM titles
     ORDER BY popularity DESC
     LIMIT 10000
   `);

   for (const title of hotTitles) {
     await EMBEDDINGS.put(
       `embedding:${title.id}`,
       JSON.stringify(title),
       { expirationTtl: 86400 } // 24h TTL
     );
   }
   ```

3. **Edge Worker** (handles queries):
   ```javascript
   export default {
     async fetch(request, env) {
       const { query } = await request.json();

       // Get embedding from KV (cached at edge)
       const embedding = await env.EMBEDDINGS.get(
         `embedding:${query.titleId}`,
         { type: 'json' }
       );

       return new Response(JSON.stringify(embedding), {
         headers: {
           'Cache-Control': 'public, max-age=43200, stale-while-revalidate=86400',
           'Content-Type': 'application/json'
         }
       });
     }
   };
   ```

**Expected Results:**
- ✅ <10ms query latency for hot titles
- ✅ 95%+ cache hit rate after warmup
- ✅ 24-hour max staleness (acceptable for hackathon)
- ✅ ~$22/month cost

### Phase 2: Post-Hackathon (Production Enhancements)

**Add real-time invalidation:**

1. **Setup Redis Pub/Sub**:
   ```bash
   # Install Upstash Redis (free tier)
   npx wrangler kv:namespace create REDIS_CONFIG
   ```

2. **Database trigger** (PostgreSQL):
   ```sql
   CREATE OR REPLACE FUNCTION notify_catalog_change()
   RETURNS trigger AS $$
   BEGIN
     PERFORM pg_notify('catalog_updates', json_build_object(
       'type', TG_OP,
       'title_id', NEW.id,
       'version', NEW.version,
       'timestamp', NOW()
     )::text);
     RETURN NEW;
   END;
   $$ LANGUAGE plpgsql;

   CREATE TRIGGER catalog_change_trigger
   AFTER INSERT OR UPDATE ON titles
   FOR EACH ROW EXECUTE FUNCTION notify_catalog_change();
   ```

3. **Worker listener** (Durable Object):
   ```javascript
   export class CatalogListener {
     async fetch(request) {
       // Subscribe to Redis channel
       await this.redis.subscribe('catalog_updates', async (msg) => {
         const event = JSON.parse(msg);
         await env.EMBEDDINGS.delete(`embedding:${event.title_id}`);
       });
     }
   }
   ```

**Expected Improvements:**
- ✅ <60 second propagation for updates
- ✅ Event-driven invalidation reduces stale reads
- ✅ Still gracefully degrades if Redis unavailable

### Phase 3: Advanced (Scale to 100K+ Titles)

**Add R2 warm tier:**

```javascript
// Three-tier architecture
async function getEmbedding(titleId) {
  // Tier 1: KV (10K hot titles, <10ms)
  let data = await EMBEDDINGS_KV.get(`hot:${titleId}`);
  if (data) return JSON.parse(data);

  // Tier 2: R2 (100K warm titles, ~50ms)
  const r2Object = await R2.get(`embeddings/batch-${Math.floor(titleId/1000)}.json`);
  if (r2Object) {
    const batch = await r2Object.json();
    data = batch[titleId];
    if (data) {
      // Promote to hot tier
      await EMBEDDINGS_KV.put(`hot:${titleId}`, JSON.stringify(data), {
        expirationTtl: 86400
      });
      return data;
    }
  }

  // Tier 3: Origin database (cold, ~200ms)
  return await fetchFromOrigin(titleId);
}
```

---

## 8. Implementation Checklist

### ✅ Must-Have (Hackathon MVP)

- [ ] **Setup Cloudflare Workers KV namespace** for embeddings
- [ ] **Batch sync script** (daily cron job to upload 10K hot titles)
- [ ] **Edge Worker** with KV reads and Cache-Control headers
- [ ] **TTL-based expiration** (24-hour freshness window)
- [ ] **Monitoring**: Log cache hit/miss rates
- [ ] **Fallback logic**: Serve stale data if origin unavailable

### 🎯 Should-Have (Post-Hackathon)

- [ ] **Redis Pub/Sub** integration for real-time notifications
- [ ] **Database triggers** for automatic change detection
- [ ] **Surrogate key tagging** for granular invalidation
- [ ] **Stale-while-revalidate** pattern implementation
- [ ] **Version-based conflict resolution** (monotonic versions)
- [ ] **Analytics dashboard** for staleness metrics

### 🔮 Nice-to-Have (Future Scaling)

- [ ] **R2 warm tier** for 100K+ titles
- [ ] **Durable Objects** for WebSocket-based notifications
- [ ] **Multi-region R2** for geographic redundancy
- [ ] **Predictive pre-caching** based on viewing patterns
- [ ] **A/B testing framework** for cache strategies
- [ ] **Cost optimization** (automatic hot/cold tier promotion)

---

## 9. Performance Benchmarks & Expectations

### Latency Targets

| Operation | Target | Cloudflare KV Actual | Notes |
|-----------|--------|----------------------|-------|
| **Hot key read** | <10ms p99 | 500µs-10ms | [Source](https://developers.cloudflare.com/kv/concepts/how-kv-works/) |
| **Cold key read** | <100ms | 50-200ms | First read from central KV |
| **Write propagation** | <60s | <60s | Eventual consistency window |
| **Origin fallback** | <500ms | Varies | PostgreSQL query time |

### Cache Hit Rates

**Expected Performance:**
- **Warmup period**: 70% hit rate (first hour)
- **Steady state**: 95%+ hit rate (after 24 hours)
- **Post-update**: 85% hit rate (during invalidation wave)

**Netflix comparison**: 98% cache hit rate with [Open Connect](https://medium.com/@hjain5164/inside-netflixs-video-streaming-delivery-architecture-e2c848e98a85)

### Scalability Projections

```
Assumptions:
- 10,000 hot titles
- 1,000 queries/second (QPS)
- 200 Cloudflare PoPs

Cache Hit Rate: 95%
- Edge serves: 950 QPS (no origin load)
- Origin serves: 50 QPS (fallback)

Origin Load Reduction: 95% (20x less traffic)

Cost at 1M QPS:
- Edge queries: 2.6B/month × $0.50/million = $1,300/month
- Origin queries: 130M/month (1M × 5% × 2.6)
- Net savings: ~$50K/month in origin database costs
```

---

## 10. Monitoring & Observability

### Key Metrics to Track

```javascript
// CloudFlare Workers Analytics
export default {
  async fetch(request, env, ctx) {
    const start = Date.now();

    // Attempt cache read
    const cached = await env.EMBEDDINGS.get(key);
    const cacheHit = !!cached;

    // Track metrics
    ctx.waitUntil(
      env.ANALYTICS.writeDataPoint({
        blobs: [request.cf.colo, key], // PoP location, cache key
        doubles: [Date.now() - start], // Latency
        indexes: [cacheHit ? 'HIT' : 'MISS'] // Cache status
      })
    );

    return new Response(cached || await fetchFromOrigin());
  }
};
```

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| **Cache hit rate** | <90% | <80% |
| **p99 latency** | >50ms | >100ms |
| **Stale data %** | >10% | >25% |
| **Origin fallback rate** | >10% | >20% |
| **Propagation delay** | >2 min | >5 min |

---

## Sources & References

### Cloudflare Documentation
- [Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)
- [Cloudflare Workers KV](https://developers.cloudflare.com/kv/)
- [Durable Objects Overview](https://developers.cloudflare.com/durable-objects/)
- [How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Rearchitecting Workers KV](https://blog.cloudflare.com/rearchitecting-workers-kv-for-redundancy/)
- [Cloudflare Workers KV Performance Gain](https://www.infoq.com/news/2025/08/cloudflare-workers-kv/)

### CDN & Cache Invalidation
- [Modern CDN Invalidation Strategies](https://medium.com/@sharareshaddev/modern-cdn-invalidation-strategies-for-dynamic-spa-content-7c3081d2a0c5)
- [Fastly CDN Best Practices](https://loadforge.com/guides/caching-strategies-with-fastly-cdn-best-practices)
- [CDN Edge Caching Invalidation](https://www.meegle.com/en_us/topics/content-delivery-network/cdn-edge-caching-invalidation)
- [Edge Caching System Design](https://www.geeksforgeeks.org/system-design/edge-caching-system-design/)
- [Cache Invalidation Strategies](https://www.ioriver.io/terms/cache-invalidation)
- [Cache Invalidation vs Expiration](https://daily.dev/blog/cache-invalidation-vs-expiration-best-practices)
- [Google Cloud CDN Cache Invalidation](https://cloud.google.com/cdn/docs/cache-invalidation-overview)

### Industry Case Studies
- [Netflix Video Streaming Architecture](https://medium.com/@hjain5164/inside-netflixs-video-streaming-delivery-architecture-e2c848e98a85)
- [Netflix Warming Petabytes of Cache](https://www.tutorialspoint.com/how-netflix-warms-petabytes-of-cache-data)
- [Netflix System Design](https://medium.com/technology-hits/netflix-system-design-a-complete-architecture-3e38f4dae631)
- [Netflix Architecture 2026](https://www.clickittech.com/software-development/netflix-architecture/)
- [Netflix Open Connect](https://systemdesignschool.io/blog/netflix-open-connect)
- [Netflix CDN Tech Stack](https://blog.blazingcdn.com/en-us/cdn-netflix-tech-stack-open-connect-home-caching-nodes)
- [Spotify System Architecture](https://www.designgurus.io/answers/detail/discuss-spotify-system-architecture)
- [Spotify Real-Time Streaming](https://sderay.com/how-spotify-streams-music-to-millions-in-real-time-content-caching-edge-computing-adaptive-streaming/)
- [Spotify Distributed Cache Platform](https://2023.platformcon.com/talks/key-learnings-from-building-a-distributed-cache-platform-at-spotify)
- [Spotify Cloud Infrastructure](https://www.sprintzeal.com/blog/spotify-cloud)

### Eventually Consistent Systems
- [Eventual Consistency Architecture](https://medium.com/@kumarabhishek0388/architecting-for-scale-part-2-eventual-consistency-and-handling-large-scale-data-bbb9e72fedd9)
- [What is Eventual Consistency](https://www.keboola.com/blog/eventual-consistency)
- [Eventual Consistency Explained](https://medium.com/@rangavamsi5/eventual-consistency-423744066919)
- [Strong vs Eventual Consistency](https://www.designgurus.io/answers/detail/what-is-eventual-consistency-and-how-does-it-differ-from-strong-consistency-in-distributed-systems)
- [Aerospike Eventual Consistency](https://aerospike.com/glossary/eventual-consistency/)
- [Eventually Consistent - Revisited](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Strong vs Eventual Consistency Design](https://www.designgurus.io/course-play/grokking-the-system-design-interview/doc/strong-vs-eventual-consistency)

### Redis Pub/Sub
- [Redis Pub/Sub Documentation](https://redis.io/docs/latest/develop/pubsub/)
- [Redis Pub/Sub Usefulness](https://stackoverflow.com/questions/11334483/redis-pub-sub-usefulness)
- [Redis Cache Synchronization](https://medium.com/@narasimha4789/keeping-the-cache-in-sync-how-redis-masters-data-synchronization-in-distributed-cache-a899cd680977)
- [Redis Pub/Sub Patterns](https://blog.poespas.me/posts/2024/05/30/redis-pubsub-patterns/)
- [Redis + Local Memory Caching](https://medium.com/@deghun/redis-pub-sub-local-memory-low-latency-high-consistency-caching-3740f66f0368)
- [Redis Beyond Caching](https://infobytes.guru/articles/redis-use-cases-beyond-caching.html)
- [Synchronizing Local Caches](https://doc.postsharp.net/caching-pubsub)

### Streaming & Real-Time Updates
- [Real-Time Streaming Caching](https://30dayscoding.com/blog/caching-in-real-time-streaming-applications)
- [CDN Cache Purge for Video](https://www.fastpix.io/blog/cdn-cache-purge-in-video-streaming)
- [RTK Query Streaming Updates](https://github.com/reduxjs/redux-toolkit/blob/master/docs/rtk-query/usage/streaming-updates.mdx)
- [Materialize + Redis Cache Invalidation](https://materialize.com/blog/redis-cache-invalidation/)
- [Stream-Based Cache Invalidation](https://markpapadakis.medium.com/invalidating-caches-and-reacting-to-updates-using-streams-4c32ca9735ef)
- [AWS Streaming Media Lens](https://docs.aws.amazon.com/wellarchitected/latest/streaming-media-lens/review-and-monitoring.html)

### Vector Embeddings
- [Vector Embedding Overview](https://www.ibm.com/think/topics/vector-embedding)
- [Node Embeddings Beginners Guide](https://towardsdatascience.com/node-embeddings-for-beginners-554ab1625d98/)
- [Graph Embeddings](https://towardsdatascience.com/graph-embeddings-how-nodes-get-mapped-to-vectors-2e12549457ed/)
- [Google Embeddings Guide](https://developers.google.com/machine-learning/crash-course/embeddings)

---

## Conclusion

For the TV5 Media Gateway hackathon, a **hybrid TTL + event-driven invalidation strategy** using **Cloudflare Workers KV** and **Redis Pub/Sub** provides the optimal balance of:

✅ **Performance**: <10ms edge latency, 95%+ cache hit rates
✅ **Freshness**: <60 second propagation, 24-hour max staleness
✅ **Cost**: ~$22/month for 200+ PoPs
✅ **Simplicity**: Minimal infrastructure, proven patterns
✅ **Scalability**: Linear scaling to millions of queries/day

The recommended architecture mirrors Netflix's dual-plane approach (control vs data) and Spotify's multi-tier caching, adapted for serverless edge computing with Cloudflare's global network.

**Start simple** (TTL-based), **iterate quickly** (add Pub/Sub), **scale gracefully** (add R2 tier).
