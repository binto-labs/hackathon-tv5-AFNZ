---
status: keep
phase: complete
type: design
version: 2.0
last-updated: 2025-12-07
title: Final Architecture Design - TV5 Media Gateway (Edge-First)
agent: Architecture Team
---

# Final Architecture Design - TV5 Media Gateway

## Executive Summary

The TV5 Media Gateway is an **edge-first media discovery platform** that deploys semantic search and behavioral analytics to the user's device. Using RuVector WASM for sub-100µs vector search and real-time activity monitoring, we provide privacy-preserving personalization that works offline.

**Key Differentiators:**
- **Edge-First Intelligence**: RuVector WASM on user device (61µs latency)
- **User Activity Monitoring**: Search, linger, trailer, browse, watch, skip signals
- **Privacy by Design**: Data stays on device, user has full control
- **Offline Capable**: Full search functionality without internet
- **Collaborative Ontology**: jjohare's GMC-O for semantic reasoning

**Authoritative Source:** See `edge-user-activity-architecture.md` for detailed technical specifications.

---

## Architecture Overview

### Three-Tier Edge Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   TIER 0: USER DEVICE (RuVector WASM)                                       │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐    │
│   │  Vector Index   │  │  User Profile   │  │  Activity Collector     │    │
│   │  ─────────────  │  │  ─────────────  │  │  ─────────────────────  │    │
│   │  10K hot titles │  │  Preferences    │  │  Search queries         │    │
│   │  HNSW (61µs)    │  │  Watch history  │  │  Linger time            │    │
│   │  85MB PQ8       │  │  Psychographic  │  │  Trailer engagement     │    │
│   └─────────────────┘  └─────────────────┘  │  Browse patterns        │    │
│                                              │  Watch completion       │    │
│   Privacy: All data stays on device          │  Skip events            │    │
│   Latency: <100ms end-to-end                 └─────────────────────────┘    │
│   Works offline                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                │ Periodic Sync (WiFi/Background)
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   TIER 1: CLOUDFLARE EDGE (330+ Global PoPs)                                │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐    │
│   │  Regional Hot   │  │  Availability   │  │  Activity Aggregator    │    │
│   │  Content Index  │  │  Cache (KV)     │  │  ─────────────────────  │    │
│   │  ─────────────  │  │  ─────────────  │  │  Anonymized signals     │    │
│   │  100K titles    │  │  Platform data  │  │  Trend detection        │    │
│   │  <30ms search   │  │  Deep links     │  │  Privacy transform      │    │
│   └─────────────────┘  └─────────────────┘  └─────────────────────────┘    │
│   Latency: <30ms                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                │ Cold Path (Background Sync)
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   TIER 2: CENTRAL CLOUD (AgentDB + Ontology Reasoning)                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  AgentDB v1.6.0          │  PostgreSQL      │  Ontology Engine     │  │
│   │  ───────────────         │  ──────────      │  ────────────────    │  │
│   │  400K+ vectors           │  Full metadata   │  jjohare's GMC-O     │  │
│   │  GNN self-learning       │  JustWatch sync  │  SWRL reasoning      │  │
│   │  Hyperbolic embeddings   │  Activity trends │  Psychographic model │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   MULTI-AGENT SWARM (Claude Flow v2.7)                                      │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ CATALOG  │  │ ENRICHER │  │ PROFILE  │  │ ANALYTICS│  │ ONTOLOGY │   │
│   │ SCOUT    │  │          │  │ BUILDER  │  │          │  │ REASONER │   │
│   │ ──────── │  │ ──────── │  │ ──────── │  │ ──────── │  │ ──────── │   │
│   │ JustWatch│  │ Embeddings│  │ Clusters │  │ Trends   │  │ OWL/SWRL │   │
│   │ TMDB sync│  │ Ontology │  │ Profiles │  │ Insights │  │ Inference│   │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│   Latency: <200ms                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tier Comparison

| Aspect | Tier 0 (Device) | Tier 1 (Edge) | Tier 2 (Central) |
|--------|-----------------|---------------|------------------|
| **Location** | User's browser/device | Cloudflare PoPs (330+) | AWS/Cloud |
| **Technology** | RuVector WASM | Workers + Vectorize | AgentDB + PostgreSQL |
| **Latency** | 61µs (local) | <30ms | <200ms |
| **Coverage** | 10K hot titles | 100K regional | 400K+ full |
| **Offline** | Yes | No | No |
| **Activity** | Full tracking | Aggregated only | Trend analysis |
| **Privacy** | Local storage | Anonymized | Aggregated |

---

## Component Specifications

### 1. Tier 0: User Device (RuVector WASM)

**Purpose:** Bring intelligence to the edge for instant, private search.

#### Vector Search Engine

```typescript
// RuVector WASM Configuration
interface RuVectorConfig {
  dimensions: 384;              // all-MiniLM-L6-v2
  indexType: 'hnsw';
  quantization: 'pq8';          // 85MB for 10K vectors
  m: 16;                        // HNSW connectivity
  efConstruction: 200;
  efSearch: 50;
}

// Performance Characteristics
- Search latency: 61µs (k=10)
- Index size: 85MB (10K vectors)
- Memory budget: <128MB total
- SIMD acceleration: Yes (where supported)
```

#### User Profile Store (IndexedDB)

```typescript
interface UserProfile {
  id: string;
  device_id_hash: string;       // SHA-256, never raw

  // Preference Vectors
  genre_preferences: number[];  // Learned from activity
  mood_preferences: number[];   // Psychographic state

  // Watch History
  recent_views: WatchEvent[];   // Last 50 items
  favorites: string[];          // User-marked

  // Activity Signals
  search_history: SearchEvent[];
  linger_events: LingerEvent[];

  // Ontology Tags
  taste_cluster: 'mainstream' | 'arthouse' | 'genre_specialist';
  psychographic_state: 'seeking_comfort' | 'seeking_challenge' | 'nostalgic' | 'adventurous';

  // Privacy
  sync_consent: boolean;
  last_sync: Date;
}
```

#### Activity Collector

```typescript
interface ActivityEvent {
  event_id: string;
  session_id: string;
  device_id: string;            // Hashed on sync
  type: 'search' | 'browse' | 'linger' | 'trailer_view' | 'watch' | 'skip' | 'rating';
  timestamp: number;

  context: {
    time_of_day: 'early_morning' | 'morning' | 'afternoon' | 'evening' | 'late_night';
    device_type: 'mobile' | 'tablet' | 'desktop' | 'tv';
    network_quality: 'offline' | 'slow' | 'fast';
    is_weekend: boolean;
  };

  payload: SearchEvent | BrowseEvent | LingerEvent | TrailerEvent | WatchEvent | SkipEvent;
}
```

### 2. Tier 1: Cloudflare Edge

**Purpose:** Regional caching and availability data with privacy-preserving aggregation.

#### Workers + Vectorize Configuration

```javascript
// wrangler.toml
[[vectorize]]
binding = "VECTORIZE"
index_name = "tv5-regional-titles"
dimensions = 384
metric = "cosine"

[[kv_namespaces]]
binding = "AVAILABILITY_KV"
id = "availability-cache"

[[kv_namespaces]]
binding = "METADATA_KV"
id = "title-metadata"
```

#### Activity Aggregator

```typescript
// Privacy Transform applied before storage
interface AggregatedActivity {
  // Anonymized
  device_hash: string;          // SHA-256(device_id + salt)
  time_bucket: string;          // 15-minute buckets
  region: string;               // Country only

  // Aggregated Signals
  search_count: number;
  browse_count: number;
  avg_linger_time: number;
  trailer_views: number;

  // No PII
  // No raw queries
  // No individual viewing data
}
```

### 3. Tier 2: Central Cloud

**Purpose:** Full catalog, ontology reasoning, and trend analysis.

#### AgentDB Configuration

```typescript
import { AgentDB } from 'agentdb';

const agentdb = new AgentDB({
  dimensions: 384,
  metric: 'cosine',
  quantization: 'pq8',
  namespace: 'tv5-catalog',

  // GNN Self-Learning
  enableGNN: true,
  learningRate: 0.01,

  // Hyperbolic Embeddings (for hierarchical data)
  useHyperbolic: true,
  curvature: -1.0
});

// Performance: Sub-100µs with 150x speedup claim
```

#### PostgreSQL Schema (Hybrid)

```sql
-- Core metadata (normalized for fast queries)
CREATE TABLE titles (
  id UUID PRIMARY KEY,
  canonical_wikidata_qid VARCHAR(20) UNIQUE,
  media_type media_type_enum NOT NULL,
  tmdb_id INTEGER,
  imdb_id VARCHAR(20),
  primary_title TEXT NOT NULL,
  release_date DATE,

  -- Ontology tags (from GMC-O)
  taste_cluster VARCHAR(50),
  mood_tags TEXT[],
  context_tags TEXT[],

  -- Quality metrics
  completeness NUMERIC(3,2),
  freshness_days INTEGER,

  -- Flexible metadata
  metadata JSONB,
  provenance JSONB,

  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Activity aggregates (anonymized)
CREATE TABLE activity_trends (
  id UUID PRIMARY KEY,
  title_id UUID REFERENCES titles(id),
  time_bucket TIMESTAMPTZ,
  region VARCHAR(10),

  search_count INTEGER,
  view_count INTEGER,
  completion_rate NUMERIC(3,2),
  avg_linger_ms INTEGER,

  UNIQUE(title_id, time_bucket, region)
);
```

---

## Multi-Agent Swarm

### Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CLAUDE FLOW v2.7 ORCHESTRATION                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │ CatalogScout │───▶│   Enricher   │───▶│ProfileBuilder│          │
│  │              │    │              │    │              │          │
│  │ • JustWatch  │    │ • Embeddings │    │ • Clusters   │          │
│  │ • TMDB sync  │    │ • Ontology   │    │ • Preferences│          │
│  │ • Availability│    │ • Quality    │    │ • Psycho-    │          │
│  └──────────────┘    └──────────────┘    │   graphic    │          │
│         │                   │            └──────────────┘          │
│         │                   │                   │                   │
│         │            ┌──────▼──────┐            │                   │
│         │            │  Analytics  │◀───────────┘                   │
│         │            │             │                                │
│         │            │ • Trends    │                                │
│         │            │ • Insights  │                                │
│         │            │ • Regional  │                                │
│         │            └──────┬──────┘                                │
│         │                   │                                       │
│         │            ┌──────▼──────┐                                │
│         └───────────▶│  Ontology   │                                │
│                      │  Reasoner   │                                │
│                      │             │                                │
│                      │ • OWL/SWRL  │                                │
│                      │ • GMC-O     │                                │
│                      │ • Context   │                                │
│                      └─────────────┘                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Agent Specifications

| Agent | Input | Processing | Output |
|-------|-------|-----------|--------|
| **CatalogScout** | API endpoints (TMDB, JustWatch) | Polling, change detection | Raw catalog data |
| **Enricher** | Raw metadata | Embedding generation, ontology tagging | Enhanced metadata |
| **ProfileBuilder** | Activity signals, preferences | Clustering, vector computation | User profiles |
| **Analytics** | Aggregated activity | Trend detection, regional analysis | Insights |
| **OntologyReasoner** | GMC-O ontology, user context | OWL/SWRL inference | Contextual recommendations |

### Agent Coordination (Claude Flow)

```typescript
// Mesh topology for parallel processing
await claudeFlow.init({
  topology: 'mesh',
  maxAgents: 8,

  agents: [
    { type: 'CatalogScout', capabilities: ['tmdb', 'justwatch', 'wikidata'] },
    { type: 'Enricher', capabilities: ['embeddings', 'ontology', 'quality'] },
    { type: 'ProfileBuilder', capabilities: ['clustering', 'preferences'] },
    { type: 'Analytics', capabilities: ['trends', 'regional'] },
    { type: 'OntologyReasoner', capabilities: ['owl', 'swrl', 'gmco'] }
  ],

  memory: {
    namespace: 'tv5-gateway',
    persistence: true
  }
});
```

---

## Ontology Integration (jjohare's GMC-O)

### Full Ontology (Cloud - 18KB)

Located at: `https://github.com/jjohare/hackathon-tv5/blob/main/semantic-recommender/design/ontology/expanded-media-ontology.ttl`

```turtle
# Key Classes from GMC-O
gmc:Media rdfs:subClassOf schema:CreativeWork .
gmc:Film rdfs:subClassOf gmc:Media .
gmc:TVSeries rdfs:subClassOf gmc:Media .

gmc:PsychographicState a owl:Class .
gmc:SeekingComfort rdfs:subClassOf gmc:PsychographicState .
gmc:SeekingChallenge rdfs:subClassOf gmc:PsychographicState .
gmc:Nostalgic rdfs:subClassOf gmc:PsychographicState .
gmc:Adventurous rdfs:subClassOf gmc:PsychographicState .

gmc:TasteCluster a owl:Class .
gmc:Mainstream rdfs:subClassOf gmc:TasteCluster .
gmc:ArtHouse rdfs:subClassOf gmc:TasteCluster .
gmc:GenreSpecialist rdfs:subClassOf gmc:TasteCluster .

gmc:ViewingContext a owl:Class .
gmc:TimeOfDay a owl:Class .
gmc:SocialSetting a owl:Class .
```

### Edge Subset (Device - 3KB compressed)

```typescript
// Minimal ontology for device-level inference
const EDGE_ONTOLOGY = {
  psychographicStates: ['seeking_comfort', 'seeking_challenge', 'nostalgic', 'adventurous'],
  tasteClusters: ['mainstream', 'arthouse', 'genre_specialist'],
  timeOfDay: ['early_morning', 'morning', 'afternoon', 'evening', 'late_night'],
  socialSettings: ['solo', 'date_night', 'family', 'friends'],

  // Inference rules (simplified from SWRL)
  rules: {
    lateNightBrowsing: {
      condition: (ctx) => ctx.time === 'late_night' && ctx.browseTime > 300,
      inference: 'seeking_comfort'
    },
    weekendAfternoon: {
      condition: (ctx) => ctx.isWeekend && ctx.time === 'afternoon',
      inference: 'adventurous'
    }
  }
};
```

---

## User Activity Monitoring

### Signal Collection Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    USER ACTIVITY PIPELINE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User Interaction                                                    │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────┐                                                │
│  │ Event Capture   │ ◀── Search, Browse, Linger, Trailer,          │
│  │ (Browser)       │     Watch, Skip, Rating                        │
│  └────────┬────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │ Local Buffer    │ ◀── IndexedDB (raw events)                     │
│  │ (5 min batch)   │                                                │
│  └────────┬────────┘                                                │
│           │                                                          │
│           ├──────────────────────────────────────┐                  │
│           │                                      │                  │
│           ▼                                      ▼                  │
│  ┌─────────────────┐                    ┌─────────────────┐        │
│  │ Profile Update  │                    │ Privacy         │        │
│  │ (Local)         │                    │ Transform       │        │
│  │                 │                    │                 │        │
│  │ • Preferences   │                    │ • Hash IDs      │        │
│  │ • History       │                    │ • Time bucket   │        │
│  │ • Psychographic │                    │ • Aggregate     │        │
│  └─────────────────┘                    └────────┬────────┘        │
│           │                                      │                  │
│           │                                      ▼                  │
│           │                             ┌─────────────────┐        │
│           │                             │ Sync to Cloud   │        │
│           │                             │ (if consented)  │        │
│           │                             └─────────────────┘        │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ Personalized    │                                                │
│  │ Recommendations │                                                │
│  └─────────────────┘                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Activity Signal Schemas

```typescript
// Search Event
interface SearchEvent {
  query: string;
  filters: Record<string, any>;
  result_count: number;
  clicked_indices: number[];
  time_to_first_click_ms: number;
}

// Linger Event
interface LingerEvent {
  title_id: string;
  element_type: 'card' | 'detail' | 'trailer';
  duration_ms: number;
  scroll_depth: number;
}

// Trailer Event
interface TrailerEvent {
  title_id: string;
  duration_watched_ms: number;
  total_duration_ms: number;
  completion_rate: number;
  replayed: boolean;
  sound_enabled: boolean;
}

// Watch Event
interface WatchEvent {
  title_id: string;
  platform: string;
  start_time: Date;
  end_time: Date;
  completion_rate: number;
  pauses: number;
}

// Skip Event
interface SkipEvent {
  title_id: string;
  skip_point_ms: number;
  reason?: 'not_interested' | 'seen_before' | 'wrong_mood' | 'too_long';
}
```

---

## Privacy Architecture

### Data Flow with Privacy Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PRIVACY LAYERS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 0: Raw Data (Device Only)                                    │
│  ├── Full user profile                                              │
│  ├── Detailed activity events                                       │
│  ├── Search queries                                                 │
│  └── Watch history                                                  │
│      ⬇ NEVER leaves device unless consented                        │
│                                                                      │
│  Layer 1: Anonymized (Edge Sync)                                    │
│  ├── device_id → SHA-256(device_id + daily_salt)                   │
│  ├── timestamp → 15-minute buckets                                  │
│  ├── location → country only                                        │
│  └── queries → excluded (never synced)                              │
│      ⬇ Only if user consents                                       │
│                                                                      │
│  Layer 2: Aggregated (Cloud)                                        │
│  ├── Regional trends only                                           │
│  ├── No individual profiles                                         │
│  └── Statistical aggregates                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### User Controls

```typescript
interface PrivacySettings {
  // Data Collection
  collectActivity: boolean;          // Default: true
  collectSearchQueries: boolean;     // Default: false

  // Data Sync
  syncToCloud: boolean;              // Default: false
  syncFrequency: 'realtime' | 'daily' | 'manual';

  // Data Retention
  localRetentionDays: number;        // Default: 90

  // User Actions
  exportData(): Promise<Blob>;
  deleteAllData(): Promise<void>;
  viewCollectedData(): Promise<ActivitySummary>;
}
```

---

## Data Flow

### Query Flow (Device-First)

```
User Query: "Find a cozy rom-com"
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: Device Search (61µs)                                       │
│  ├── Generate embedding (cached model)                              │
│  ├── RuVector WASM HNSW search (10K titles)                        │
│  ├── Apply user preferences                                         │
│  └── Return top 20 matches                                          │
│                                                                      │
│  IF sufficient results (>5 with score >0.7):                        │
│      ✓ Return immediately (<100ms total)                            │
│  ELSE:                                                              │
│      ▼                                                              │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 2: Edge Search (<30ms)                                        │
│  ├── Forward to nearest Cloudflare PoP                              │
│  ├── Vectorize search (100K regional titles)                        │
│  ├── Merge with device results                                      │
│  └── Enrich with availability data                                  │
│                                                                      │
│  IF sufficient results:                                              │
│      ✓ Return (<130ms total)                                        │
│  ELSE:                                                              │
│      ▼                                                              │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 3: Central Search (<200ms)                                    │
│  ├── AgentDB full catalog search                                    │
│  ├── Ontology reasoning (GMC-O)                                     │
│  ├── Advanced filtering                                             │
│  └── Quality scoring                                                │
│                                                                      │
│  ✓ Return complete results (<330ms total)                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Activity Sync Flow

```
User Activity (continuous)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Local Collection (always)                                          │
│  ├── Buffer events in memory                                        │
│  ├── Batch write to IndexedDB every 5 minutes                      │
│  ├── Update local profile immediately                               │
│  └── Apply psychographic inference                                  │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼ (if sync enabled)
┌─────────────────────────────────────────────────────────────────────┐
│  Privacy Transform                                                  │
│  ├── Hash device ID                                                 │
│  ├── Bucket timestamps (15-min)                                     │
│  ├── Remove PII (queries, exact titles)                            │
│  └── Aggregate signals                                              │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Edge Aggregation                                                   │
│  ├── Receive anonymized signals                                     │
│  ├── Regional trend detection                                       │
│  ├── Forward aggregates to central                                  │
│  └── Cache popular content signals                                  │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Central Analytics                                                  │
│  ├── Store aggregated trends                                        │
│  ├── Update hot title list                                          │
│  ├── Feed into agent swarm                                          │
│  └── No individual user data                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Device Vector** | RuVector WASM | 61µs HNSW search |
| **Device Storage** | IndexedDB | Profiles, activity, cache |
| **Device Embeddings** | all-MiniLM-L6-v2 | 384-dim, pre-computed |
| **Edge Compute** | Cloudflare Workers | 330+ PoPs, <30ms |
| **Edge Vector** | Vectorize | 100K regional titles |
| **Edge Cache** | Workers KV | Availability, metadata |
| **Central Vector** | AgentDB v1.6.0 | 400K+ full catalog |
| **Central DB** | PostgreSQL + JSONB | Metadata, provenance |
| **Ontology** | jjohare's GMC-O | OWL/SWRL reasoning |
| **Orchestration** | Claude Flow v2.7 | Agent coordination |
| **Data Sources** | TMDB, JustWatch, Wikidata | Catalog, availability |

---

## Hackathon MVP Scope

### Must Have (Demo)

- [ ] RuVector WASM deployed to demo app
- [ ] 10K hot titles indexed locally
- [ ] Basic activity collection (search, linger)
- [ ] Psychographic state inference
- [ ] Privacy dashboard
- [ ] Offline mode working
- [ ] Speed comparison (local vs cloud)

### Nice to Have

- [ ] Full 6 activity signals
- [ ] jjohare ontology integration
- [ ] Edge tier deployment
- [ ] Profile export functionality

### Defer to Post-Hackathon

- Tier 1 (Cloudflare) full deployment
- Federated learning
- Smart TV apps
- Production privacy audit

---

## Performance Targets

| Metric | Target | Tier |
|--------|--------|------|
| **Local Search** | 61µs | Tier 0 |
| **Edge Search** | <30ms | Tier 1 |
| **Central Search** | <200ms | Tier 2 |
| **Activity Overhead** | <1ms | Tier 0 |
| **Profile Update** | <10ms | Tier 0 |
| **Offline Available** | 100% | Tier 0 |
| **Memory Budget** | <128MB | Tier 0 |

---

## References

- `edge-user-activity-architecture.md` - SOURCE OF TRUTH
- `README.md` (fork root) - Project overview
- `CONTEXT-REHYDRATION.md` - Strategic decisions
- jjohare's GMC-O: https://github.com/jjohare/hackathon-tv5

---

**Document Version:** 2.0 (Edge-First)
**Last Updated:** 2025-12-07
**Status:** Ready for implementation
