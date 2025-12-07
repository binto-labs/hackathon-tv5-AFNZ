---
status: keep
phase: active
type: architecture
version: 1.0
last-updated: 2025-12-07
title: Edge-First User Activity Architecture with RuVector/WASM
author: Architecture Swarm (TV5 Media Gateway)
tags: [edge, ruvector, wasm, user-activity, ontology, architecture]
---

# Edge-First User Activity Architecture

## Executive Summary

This document defines the **edge-first architecture** for TV5 Media Gateway, focusing on:
1. **RuVector/WASM deployment at the edge** for sub-100ms semantic search
2. **Comprehensive user activity monitoring** at the edge (not just search, but behavior)
3. **Ontology adoption** - Collaborating with jjohare's Global Media & Context Ontology (GMC-O)
4. **Catalog search and update mechanism** for streaming availability

**Strategic Decision**: We are the **only team doing edge-deployed semantic search** with user activity monitoring. This is our differentiator. We double down on edge.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           USER DEVICE (EDGE)                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    RuVector WASM Runtime                                 │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │    │
│  │  │ Vector Index  │  │ User Profile  │  │ Activity      │               │    │
│  │  │ (Hot 10K)     │  │ (Local State) │  │ Collector     │               │    │
│  │  │ ─────────────│  │ ─────────────│  │ ─────────────│               │    │
│  │  │ HNSW k=10    │  │ Preferences  │  │ Search terms │               │    │
│  │  │ 61µs search  │  │ Watch history│  │ Linger time  │               │    │
│  │  │ 85MB PQ8     │  │ Psychographic│  │ Trailer views│               │    │
│  │  └───────────────┘  └───────────────┘  └───────────────┘               │    │
│  │                           │                                              │    │
│  │              ┌────────────▼────────────┐                                │    │
│  │              │   Edge Query Engine      │                                │    │
│  │              │   ────────────────────   │                                │    │
│  │              │   • Semantic search      │                                │    │
│  │              │   • Context weighting    │                                │    │
│  │              │   • Personalization      │                                │    │
│  │              │   • Activity analytics   │                                │    │
│  │              └────────────┬────────────┘                                │    │
│  └───────────────────────────┼──────────────────────────────────────────────┘    │
│                              │                                                    │
│  ┌───────────────────────────▼──────────────────────────────────────────────┐    │
│  │                    Local Activity Store (IndexedDB)                       │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │    │
│  │  │ Search Log  │  │ Browse Log  │  │ View Log    │  │ Sync Queue  │     │    │
│  │  │ ──────────  │  │ ──────────  │  │ ──────────  │  │ ──────────  │     │    │
│  │  │ Queries     │  │ Linger time │  │ Watch time  │  │ Pending     │     │    │
│  │  │ Timestamps  │  │ Genre paths │  │ Completion  │  │ uploads     │     │    │
│  │  │ Results     │  │ Scroll depth│  │ Skip events │  │ compressed  │     │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ Periodic Sync (WiFi/Low-Battery)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        CLOUDFLARE EDGE (330+ PoPs)                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    Cloudflare Workers + Vectorize                        │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │    │
│  │  │ Hot Index     │  │ Activity      │  │ Sync Engine   │               │    │
│  │  │ (100K titles) │  │ Aggregator    │  │               │               │    │
│  │  │ ─────────────│  │ ─────────────│  │ ─────────────│               │    │
│  │  │ Regional hot │  │ Batch writes  │  │ Delta sync    │               │    │
│  │  │ content      │  │ Anonymization │  │ Conflict res  │               │    │
│  │  │ <30ms search │  │ Aggregation   │  │ Compression   │               │    │
│  │  └───────────────┘  └───────────────┘  └───────────────┘               │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ Cold Path (Background)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        CENTRAL CLOUD                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  RuVector/AgentDB v1.6.0          │  PostgreSQL + Redis                  │    │
│  │  ───────────────────────          │  ─────────────────                   │    │
│  │  400K+ title vectors              │  Full metadata store                 │    │
│  │  Sub-100µs similarity             │  Streaming availability              │    │
│  │  GNN self-learning               │  User activity analytics             │    │
│  │  Hyperbolic embeddings            │  Ontology triple store               │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    Multi-Agent Swarm (Claude Flow)                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ CATALOG  │  │ ENRICHER │  │ PROFILE  │  │ ANALYTICS│  │ ONTOLOGY │ │    │
│  │  │ SCOUT    │  │          │  │ BUILDER  │  │          │  │ REASONER │ │    │
│  │  │ ──────── │  │ ──────── │  │ ──────── │  │ ──────── │  │ ──────── │ │    │
│  │  │ JustWatch│  │ Add      │  │ Build    │  │ Trend    │  │ SWRL/OWL │ │    │
│  │  │ TMDB API │  │ context  │  │ profiles │  │ analysis │  │ inference│ │    │
│  │  │ IMDb sync│  │ embeddings│  │ clusters │  │ insights │  │ reasoning│ │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. User Activity Monitoring Signals

### 2.1 Signal Categories

We capture **behavioral intelligence** at the edge, not just search queries:

| Signal Category | Specific Metrics | Edge Storage | Sync Frequency |
|----------------|------------------|--------------|----------------|
| **Search Behavior** | Query text, filters applied, results clicked | IndexedDB | Real-time local, batch sync |
| **Linger Time** | Time on result card (hover/focus), scroll velocity | IndexedDB | Session end |
| **Trailer Engagement** | View duration, completion %, replay count | IndexedDB | Session end |
| **Browse Patterns** | Genre navigation path, category dwell time | IndexedDB | Session end |
| **Watch History** | Title, start/stop times, completion % | IndexedDB | On wifi |
| **Skip Events** | What was skipped, at what point, reason (if UI offers) | IndexedDB | Batch |
| **Context Signals** | Time of day, device type, network quality | Memory | Per request |

### 2.2 Activity Collection Schema

```typescript
// Edge Activity Event Schema
interface ActivityEvent {
  // Identity
  event_id: string;          // UUID v7 (time-ordered)
  session_id: string;        // Session identifier
  device_id: string;         // Hashed device fingerprint

  // Event Type
  type: 'search' | 'browse' | 'linger' | 'trailer_view' | 'watch' | 'skip' | 'rating';

  // Timestamp
  timestamp: number;         // Unix ms
  timezone_offset: number;   // Local timezone

  // Context (auto-captured)
  context: {
    time_of_day: 'morning' | 'afternoon' | 'evening' | 'late_night';
    device_type: 'mobile' | 'tablet' | 'tv' | 'desktop';
    network_quality: 'slow' | 'medium' | 'fast' | 'offline';
    is_weekend: boolean;
  };

  // Event-specific payload
  payload: SearchEvent | BrowseEvent | LingerEvent | TrailerEvent | WatchEvent | SkipEvent | RatingEvent;
}

interface SearchEvent {
  query: string;
  filters: Record<string, string[]>;  // genre, year, platform filters
  result_count: number;
  results_shown: string[];            // Title IDs shown
  result_clicked: string | null;      // Title ID clicked
  time_to_click_ms: number | null;
}

interface LingerEvent {
  title_id: string;
  linger_duration_ms: number;
  interaction: 'hover' | 'focus' | 'expand';
  scroll_position: number;            // 0-100%
  next_action: 'click' | 'scroll_away' | 'back';
}

interface TrailerEvent {
  title_id: string;
  trailer_duration_sec: number;
  watched_duration_sec: number;
  completion_pct: number;
  replays: number;
  sound_on: boolean;
  fullscreen: boolean;
  exit_action: 'click_watch' | 'close' | 'scroll_away';
}

interface WatchEvent {
  title_id: string;
  platform: string;                   // Netflix, Disney+, etc.
  start_timestamp: number;
  end_timestamp: number;
  watch_duration_sec: number;
  total_duration_sec: number;
  completion_pct: number;
  pauses: number;
  seeking_events: number;
}

interface SkipEvent {
  title_id: string;
  skip_point_pct: number;             // Where in content they skipped
  skip_reason?: 'boring' | 'too_intense' | 'not_what_expected' | 'other';
  context: 'search_result' | 'recommendation' | 'browse';
}

interface BrowseEvent {
  path: string[];                     // Genre navigation path
  dwell_times: Record<string, number>; // ms per category
  scroll_depth_pct: number;
  items_viewed: number;
  items_interacted: number;
}
```

### 2.3 Privacy-First Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRIVACY ARCHITECTURE                          │
│                                                                  │
│  EDGE (User Device)                                              │
│  ─────────────────                                               │
│  • Raw events stored locally in IndexedDB                        │
│  • Full resolution data for local personalization                │
│  • User has full control (export, delete)                        │
│  • No PII leaves device without consent                          │
│                                                                  │
│  SYNC TRANSFORM                                                  │
│  ──────────────                                                  │
│  • Device ID → SHA-256 hash (non-reversible)                     │
│  • Timestamps → Bucketed to 15-minute windows                    │
│  • Location → Country-level only (from IP at edge)               │
│  • Query text → Embedding vector only (no raw text)              │
│                                                                  │
│  CENTRAL CLOUD                                                   │
│  ─────────────                                                   │
│  • Receives anonymized, aggregated signals                       │
│  • Cannot reconstruct individual browsing history                │
│  • Used for trend analysis and model training                    │
│  • GDPR/CCPA compliant by design                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Ontology Adoption: Global Media & Context Ontology (GMC-O)

### 3.1 Collaboration with jjohare

**Strategy**: This is a **collaborative hackathon**. We adopt jjohare's comprehensive GMC-O ontology and extend it for edge deployment.

**jjohare's GMC-O provides**:
- Full RDF/OWL 2.0 ontology with 50+ properties
- Multi-modal embeddings (1024-dim unified)
- Psychographic user states
- Context awareness (time, device, social setting)
- Semantic reasoning rules (SWRL)

**We contribute**:
- Edge-optimized subset for WASM deployment
- Real-time user activity signal integration
- Streaming availability tracking
- Behavioral analytics feedback loop

### 3.2 Ontology Subset for Edge

The full GMC-O is 18KB+ and includes reasoning rules not needed at edge. We deploy a **compressed subset**:

```turtle
# Edge Ontology Subset (3KB compressed)

# Core Media Classes (minimal hierarchy)
media:CreativeWork a owl:Class .
media:Film rdfs:subClassOf media:CreativeWork .
media:TVSeries rdfs:subClassOf media:CreativeWork .

# Psychographic States (for personalization)
user:PsychographicState a owl:Class .
user:SeekingComfort rdfs:subClassOf user:PsychographicState .
user:SeekingChallenge rdfs:subClassOf user:PsychographicState .
user:Nostalgic rdfs:subClassOf user:PsychographicState .
user:Adventurous rdfs:subClassOf user:PsychographicState .

# Context Classes (for query weighting)
ctx:TimeOfDay a owl:Class .
ctx:EarlyMorning rdfs:subClassOf ctx:TimeOfDay .
ctx:LateNight rdfs:subClassOf ctx:TimeOfDay .

ctx:SocialSetting a owl:Class .
ctx:DateNight rdfs:subClassOf ctx:SocialSetting .
ctx:FamilyGathering rdfs:subClassOf ctx:SocialSetting .
ctx:SoloViewing rdfs:subClassOf ctx:SocialSetting .

# Mood Classes (for result ranking)
media:Mood a owl:Class .
media:Comforting rdfs:subClassOf media:Mood .
media:Thrilling rdfs:subClassOf media:Mood .
media:Contemplative rdfs:subClassOf media:Mood .

# Key Properties (edge-relevant only)
sem:inducesPsychographicState a owl:ObjectProperty ;
    rdfs:domain media:CreativeWork ;
    rdfs:range user:PsychographicState .

ctx:suitableForTimeOfDay a owl:ObjectProperty ;
    rdfs:domain media:CreativeWork ;
    rdfs:range ctx:TimeOfDay .

ctx:suitableForSocialSetting a owl:ObjectProperty ;
    rdfs:domain media:CreativeWork ;
    rdfs:range ctx:SocialSetting .
```

### 3.3 User Profile Model (Edge)

```typescript
// Edge User Profile (RuVector-compatible)
interface EdgeUserProfile {
  // Identity (hashed)
  profile_id: string;

  // Current Psychographic State (inferred from activity)
  current_state: 'seeking_comfort' | 'seeking_challenge' | 'nostalgic' | 'adventurous';
  state_confidence: number;  // 0.0-1.0

  // Taste Cluster (from central, cached)
  taste_cluster: 'mainstream' | 'art_house' | 'genre_specialist';

  // Tolerance Levels (learned from behavior)
  tolerances: {
    violence: number;       // 0.0-1.0
    complexity: number;     // 0.0-1.0
    subtitles: number;      // 0.0-1.0
    slow_pacing: number;    // 0.0-1.0
  };

  // Preference Embedding (384-dim, matches search index)
  preference_vector: Float32Array;

  // Recent History (for context)
  recent_watches: RecentWatch[];  // Last 20
  recent_searches: RecentSearch[]; // Last 50

  // Session State
  session: {
    start_time: number;
    queries_count: number;
    browse_time_ms: number;
    inferred_intent: 'exploring' | 'deciding' | 'rewatching' | 'unknown';
  };
}

interface RecentWatch {
  title_id: string;
  timestamp: number;
  completion_pct: number;
  mood_tags: string[];  // From ontology
}

interface RecentSearch {
  query_embedding: Float32Array;  // Not raw text
  timestamp: number;
  result_selected: string | null;
}
```

---

## 4. RuVector/WASM Edge Deployment

### 4.1 WASM Bundle Configuration

```typescript
// RuVector WASM Configuration for TV5
const ruvectorConfig = {
  // Index Settings
  index: {
    type: 'hnsw',
    dimensions: 384,          // all-MiniLM-L6-v2
    M: 16,                    // HNSW connectivity
    efConstruction: 100,
    efSearch: 50,
    maxElements: 10000,       // Hot titles for edge
  },

  // Quantization for Memory Efficiency
  quantization: {
    type: 'pq8',              // Product Quantization 8-bit
    memoryReduction: 8,       // 8x smaller
    accuracyLoss: 0.02,       // ~2% recall loss (acceptable)
  },

  // WASM Runtime Settings
  wasm: {
    simd: true,               // Enable SIMD if available
    threads: false,           // SharedArrayBuffer not always available
    memoryLimit: 128 * 1024 * 1024,  // 128MB max (Cloudflare limit)
  },

  // Memory Budget
  memoryBudget: {
    vectorIndex: 85 * 1024 * 1024,    // 85MB for 10K vectors (PQ8)
    userProfile: 5 * 1024 * 1024,     // 5MB for profile + history
    activityLog: 10 * 1024 * 1024,    // 10MB for pending sync
    runtime: 28 * 1024 * 1024,        // 28MB for WASM runtime
  }
};
```

### 4.2 Edge Query Flow

```typescript
// Edge Search with Activity Tracking
async function edgeSearch(query: string, userProfile: EdgeUserProfile): Promise<SearchResult[]> {
  const startTime = performance.now();

  // 1. Generate query embedding (cached model, ~50ms first time, ~5ms cached)
  const queryEmbedding = await embedQuery(query);

  // 2. Context-aware query modification
  const contextVector = buildContextVector(userProfile, getCurrentContext());
  const enhancedQuery = weightedCombine(queryEmbedding, contextVector, 0.8, 0.2);

  // 3. RuVector WASM search (~61µs)
  const candidates = await ruvector.search(enhancedQuery, {
    k: 50,  // Get 50 candidates
    filter: buildAvailabilityFilter(userProfile.platforms),
  });

  // 4. Ontology-based re-ranking
  const reranked = rerankedByOntology(candidates, userProfile);

  // 5. Track activity (async, non-blocking)
  trackSearchEvent({
    query,
    queryEmbedding,
    results: reranked.slice(0, 20).map(r => r.id),
    latencyMs: performance.now() - startTime,
  });

  return reranked.slice(0, 20);
}

// Ontology-based re-ranking
function rerankedByOntology(
  candidates: SearchCandidate[],
  profile: EdgeUserProfile
): SearchCandidate[] {
  return candidates.map(candidate => {
    let score = candidate.similarity;

    // Boost based on psychographic match
    if (candidate.induces_state === profile.current_state) {
      score *= 1.3;
    }

    // Boost based on context
    const context = getCurrentContext();
    if (context.time_of_day === 'late_night' && candidate.suitable_for_late_night) {
      score *= 1.2;
    }
    if (context.is_weekend && candidate.duration > 120) {
      score *= 1.1;  // Boost longer content on weekends
    }

    // Penalize based on tolerance mismatch
    if (candidate.violence_level > profile.tolerances.violence) {
      score *= 0.5;
    }

    return { ...candidate, score };
  }).sort((a, b) => b.score - a.score);
}
```

### 4.3 Profile Update from Activity

```typescript
// Real-time profile update from user activity
function updateProfileFromActivity(
  profile: EdgeUserProfile,
  event: ActivityEvent
): EdgeUserProfile {
  const updated = { ...profile };

  switch (event.type) {
    case 'linger':
      // Long linger = interest signal
      if (event.payload.linger_duration_ms > 5000) {
        const title = getTitleMetadata(event.payload.title_id);
        // Move preference vector toward this content
        updated.preference_vector = moveToward(
          profile.preference_vector,
          title.embedding,
          0.01  // Small learning rate
        );

        // Update psychographic state inference
        if (title.induces_state) {
          updated.current_state = inferStateFromRecent(profile, title.induces_state);
        }
      }
      break;

    case 'trailer_view':
      if (event.payload.completion_pct > 80) {
        // High trailer completion = strong interest
        const title = getTitleMetadata(event.payload.title_id);
        updated.preference_vector = moveToward(
          profile.preference_vector,
          title.embedding,
          0.02  // Stronger signal
        );
      }
      break;

    case 'skip':
      // Skip = negative signal
      const skippedTitle = getTitleMetadata(event.payload.title_id);
      updated.preference_vector = moveAway(
        profile.preference_vector,
        skippedTitle.embedding,
        0.01
      );

      // Update tolerance if skip was due to content
      if (event.payload.skip_reason === 'too_intense') {
        updated.tolerances.violence *= 0.95;
      }
      break;

    case 'watch':
      if (event.payload.completion_pct > 70) {
        // Completed watch = very strong positive signal
        const title = getTitleMetadata(event.payload.title_id);
        updated.preference_vector = moveToward(
          profile.preference_vector,
          title.embedding,
          0.05  // Strong learning
        );

        // Update tolerances based on what was tolerated
        if (title.violence_level > 0.7) {
          updated.tolerances.violence = Math.min(
            updated.tolerances.violence * 1.05,
            1.0
          );
        }
      }
      break;
  }

  // Save updated profile locally
  saveProfile(updated);

  return updated;
}
```

---

## 5. Catalog Search and Update Mechanism

### 5.1 Challenge: Fresh Streaming Availability

The hardest problem: **What's actually available to stream RIGHT NOW?**

- Netflix adds/removes 100+ titles monthly
- Regional availability differs (US vs EU vs APAC)
- Licensing changes with no notice
- User has multiple subscriptions to check

### 5.2 Solution: Tiered Availability Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CATALOG AVAILABILITY TIERS                               │
│                                                                              │
│  TIER 1: User's Platforms (Edge, Real-time)                                 │
│  ─────────────────────────────────────────                                   │
│  • User's actual subscriptions stored in profile                             │
│  • Availability checked at query time via cached mapping                     │
│  • Updated every session start (background fetch)                            │
│  • Example: "Show me what I can watch on MY Netflix + Disney+"               │
│                                                                              │
│  TIER 2: Regional Hot Content (Cloudflare Edge, Hourly)                     │
│  ──────────────────────────────────────────────────────                      │
│  • Top 10K titles per region with platform availability                      │
│  • Updated hourly from JustWatch aggregated feed                             │
│  • Stored in Cloudflare KV (key = region + title_id)                        │
│  • Example: "What's trending and where to watch it"                          │
│                                                                              │
│  TIER 3: Full Catalog (Central, Daily)                                       │
│  ─────────────────────────────────────                                       │
│  • 400K+ titles with full availability matrix                                │
│  • Daily sync from TMDB + JustWatch + IMDb                                   │
│  • Stored in PostgreSQL with materialized views                              │
│  • Example: "Find any movie ever made and where to watch"                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Catalog Update Pipeline

```typescript
// Catalog Scout Agent (Claude Flow Swarm)
const CatalogScoutAgent = {
  name: 'catalog-scout',
  type: 'researcher',
  schedule: '0 * * * *',  // Hourly

  async run() {
    // 1. Fetch changes from JustWatch API
    const changes = await justwatch.getChanges({
      since: lastSyncTime,
      regions: ['US', 'GB', 'CA', 'AU', 'DE', 'FR'],
      types: ['movie', 'show'],
    });

    // 2. Process additions
    for (const addition of changes.added) {
      // Update availability in PostgreSQL
      await db.updateAvailability({
        title_id: addition.title_id,
        platform: addition.platform,
        region: addition.region,
        available_since: addition.date,
        deep_link: addition.url,
      });

      // If hot content, push to edge
      if (await isHotContent(addition.title_id)) {
        await pushToEdge(addition);
      }
    }

    // 3. Process removals
    for (const removal of changes.removed) {
      await db.markUnavailable({
        title_id: removal.title_id,
        platform: removal.platform,
        region: removal.region,
        unavailable_since: removal.date,
      });

      // Remove from edge if no longer available anywhere
      if (await shouldRemoveFromEdge(removal.title_id)) {
        await removeFromEdge(removal.title_id);
      }
    }

    // 4. Push updated hot content to Cloudflare KV
    await syncHotContentToEdge();

    return { added: changes.added.length, removed: changes.removed.length };
  }
};
```

### 5.4 Edge Availability Check

```typescript
// Fast availability check at edge
async function checkAvailability(
  titleId: string,
  userPlatforms: string[],
  region: string
): Promise<AvailabilityResult> {
  // 1. Check local cache first (user's recent lookups)
  const cached = localAvailabilityCache.get(`${titleId}-${region}`);
  if (cached && cached.timestamp > Date.now() - 3600000) {  // 1 hour TTL
    return filterByUserPlatforms(cached.availability, userPlatforms);
  }

  // 2. Check Cloudflare KV (regional hot content)
  const edgeData = await AVAILABILITY_KV.get(`${region}:${titleId}`, 'json');
  if (edgeData) {
    localAvailabilityCache.set(`${titleId}-${region}`, {
      availability: edgeData,
      timestamp: Date.now(),
    });
    return filterByUserPlatforms(edgeData, userPlatforms);
  }

  // 3. Fallback to central API (cache miss)
  const centralData = await fetch(`${CENTRAL_API}/availability/${titleId}?region=${region}`);
  const availability = await centralData.json();

  // Cache locally for future
  localAvailabilityCache.set(`${titleId}-${region}`, {
    availability,
    timestamp: Date.now(),
  });

  return filterByUserPlatforms(availability, userPlatforms);
}

interface AvailabilityResult {
  available: boolean;
  platforms: {
    name: string;
    type: 'subscription' | 'rent' | 'buy';
    price?: number;
    deepLink: string;
  }[];
  leaving_soon?: {
    platform: string;
    date: string;
  };
}
```

---

## 6. Performance Targets

### 6.1 Latency Budget

| Operation | Target | Edge (WASM) | Central (Fallback) |
|-----------|--------|-------------|-------------------|
| **Semantic Search** | <100ms | 61µs | <30ms |
| **Profile Load** | <50ms | <10ms (IndexedDB) | N/A |
| **Availability Check** | <100ms | <5ms (KV) | <50ms (API) |
| **Activity Tracking** | <10ms | <1ms (async) | N/A |
| **Context Inference** | <20ms | <10ms | N/A |
| **Full Query Response** | <200ms | <150ms | <300ms |

### 6.2 Memory Budget (Edge Device)

| Component | Allocation | Notes |
|-----------|-----------|-------|
| RuVector WASM Runtime | 28MB | One-time load |
| Vector Index (10K PQ8) | 85MB | Hot titles |
| User Profile | 5MB | Preferences + history |
| Activity Log | 10MB | Pending sync queue |
| Ontology Subset | 1MB | Compressed TTL |
| **Total** | **129MB** | Under 128MB Worker limit |

### 6.3 Scale Targets

| Metric | Hackathon Demo | Production V1 | Future |
|--------|---------------|---------------|--------|
| Edge Titles | 10K | 100K | 400K+ |
| User Profiles | 50 (demo) | 10K | 1M+ |
| Events/Day | 5K | 500K | 50M+ |
| Regions | 3 | 15 | 50+ |

---

## 7. Implementation Roadmap

### Phase 1: Hackathon MVP

1. **RuVector WASM Integration**
   - Deploy ruvector-wasm bundle to Cloudflare Workers
   - Load 10K hot titles with PQ8 quantization
   - Implement basic semantic search

2. **User Activity Collection**
   - Implement IndexedDB storage for activity events
   - Track search, linger, browse events
   - Build local profile from activity

3. **Ontology Subset**
   - Extract edge-relevant classes from jjohare's GMC-O
   - Implement ontology-based re-ranking
   - Add psychographic state inference

4. **Catalog Integration**
   - JustWatch API for availability
   - Cloudflare KV for hot content
   - Basic availability filtering

### Phase 2: Demo Polish

1. **Activity Dashboard**
   - Visualize user activity signals
   - Show profile evolution over time
   - Display ontology reasoning

2. **Latency Comparison**
   - Side-by-side edge vs central
   - Activity tracking overhead visualization
   - Memory usage dashboard

3. **Collaborative Demo**
   - Link to jjohare's ontology work
   - Show complementary architectures
   - Joint presentation materials

### Phase 3: Post-Hackathon

1. **Scale to 100K+ titles at edge**
2. **Production privacy compliance**
3. **Real streaming platform integrations**
4. **Federated learning for global insights**

---

## 8. Collaboration Opportunities

### With jjohare's Team

| Their Contribution | Our Contribution |
|-------------------|------------------|
| Full GMC-O ontology (18KB TTL) | Edge-optimized subset (3KB) |
| GPU-accelerated inference | WASM edge deployment |
| 1024-dim multi-modal embeddings | Real-time activity signals |
| SWRL reasoning rules | Behavioral feedback loop |
| Psychographic modeling | Privacy-first activity tracking |

### Proposed Integration

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    COLLABORATIVE ARCHITECTURE                               │
│                                                                             │
│  jjohare (GPU Cloud)              TV5 (Edge)              Shared           │
│  ─────────────────                ───────────             ──────           │
│  • Full ontology reasoning        • Edge search           • GMC-O ontology │
│  • Multi-modal embeddings         • Activity tracking     • Embedding space│
│  • 32,000 QPS inference           • User profiles         • Wikidata QIDs  │
│  • TensorRT optimization          • <100ms latency        • Metadata schema│
│                                                                             │
│                    ┌───────────────────────────┐                           │
│                    │   COMMON API CONTRACT      │                           │
│                    │   ────────────────────     │                           │
│                    │   POST /embed              │                           │
│                    │   POST /search             │                           │
│                    │   GET  /availability       │                           │
│                    │   POST /feedback           │                           │
│                    └───────────────────────────┘                           │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Sources

### RuVector/AgentDB
- [RuVector GitHub](https://github.com/ruvnet/ruvector) - Core vector database
- [AgentDB npm](https://www.npmjs.com/package/agentdb) - Application layer
- [RuVector Validation Report](./ruvector-agentdb-validation.md) - Performance analysis

### jjohare Ontology
- [jjohare/hackathon-tv5](https://github.com/jjohare/hackathon-tv5) - Semantic recommender
- [expanded-media-ontology.ttl](https://github.com/jjohare/hackathon-tv5/blob/main/semantic-recommender/design/ontology/expanded-media-ontology.ttl) - Full ontology

### Edge Computing
- [Cloudflare Vectorize](https://developers.cloudflare.com/vectorize/) - Edge vector search
- [Cloudflare Workers KV](https://developers.cloudflare.com/workers/runtime-apis/kv/) - Edge storage
- [WASM Edge Research](./wasm-vector-research.md) - WASM performance analysis

### Data Sources
- [JustWatch API](https://www.justwatch.com/) - Streaming availability
- [TMDB API](https://developer.themoviedb.org/) - Media metadata
- [Wikidata](https://www.wikidata.org/) - Entity resolution QIDs

---

*Document Version: 1.0 | Last Updated: 2025-12-07 | Author: Architecture Swarm*
