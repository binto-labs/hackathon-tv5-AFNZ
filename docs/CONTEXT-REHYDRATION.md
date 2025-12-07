# Context Rehydration Document
## TV5 Media Gateway - Document Alignment Task

**Created**: 2025-12-07
**Purpose**: Preserve context for document alignment task after context compaction
**Status**: Ready for document alignment phase

---

## 1. Strategic Pivot Summary

### The Decision
We are **doubling down on edge-first architecture** with:
1. **RuVector WASM on user device** (not deferred - hackathon deliverable)
2. **Comprehensive user activity monitoring** at the edge
3. **Collaboration with jjohare** on GMC-O ontology
4. **Privacy-first design** - data stays on device

### Our Differentiator
> "The only hackathon team doing edge-deployed semantic search with real-time user activity monitoring and multi-source entity resolution"

While others optimize for GPU throughput (jjohare: 32K QPS), we bring intelligence to the edge with sub-100ms local search, behavioral analytics, and personalization running on the user's device.

---

## 2. New Three-Tier Architecture

```
TIER 0: USER DEVICE (RuVector WASM) ← NEW - This is our differentiator
├── Vector Index: 10K hot titles, HNSW 61µs, 85MB PQ8
├── User Profile: Preferences, watch history, psychographic state
├── Activity Collector: Search, linger, trailer, browse, watch, skip
├── Privacy: All data stays on device
├── Latency: <100ms end-to-end
└── Works offline

TIER 1: CLOUDFLARE EDGE (330+ PoPs)
├── Regional Hot Content: 100K titles
├── Availability Cache: Workers KV with platform deep links
├── Activity Aggregator: Anonymized signals, trend detection
└── Latency: <30ms

TIER 2: CENTRAL CLOUD
├── AgentDB v1.6.0: 400K+ vectors, GNN self-learning
├── PostgreSQL: Full metadata, JustWatch sync
├── Ontology Engine: jjohare's GMC-O with OWL/SWRL reasoning
└── Agent Swarm: CatalogScout, Enricher, ProfileBuilder, Analytics, OntologyReasoner
```

### Old vs New Tier Numbering
| Old Docs | New Direction |
|----------|---------------|
| Tier 1 = Cloudflare Edge | Tier 0 = User Device (WASM) |
| Tier 2 = Regional WASM (deferred) | Tier 1 = Cloudflare Edge |
| Tier 3 = Central Cloud | Tier 2 = Central Cloud |

---

## 3. User Activity Monitoring (NEW CONCEPT)

### Activity Signals Collected at Edge
| Signal | What We Track | Why It Matters |
|--------|--------------|----------------|
| **Search Behavior** | Queries, filters, clicked results | Understand intent |
| **Linger Time** | How long on each result card | Interest signal |
| **Trailer Engagement** | Watch duration, replays, sound on | Strong intent |
| **Browse Patterns** | Genre navigation, category dwell | Preference discovery |
| **Watch Completion** | Start/stop times, completion % | Satisfaction signal |
| **Skip Events** | What was skipped and when | Negative signal |

### Privacy Architecture
- Raw events stored locally in IndexedDB
- User has full control (export, delete)
- Sync transform: Device ID → SHA-256 hash, timestamps → 15-min buckets
- Central cloud receives only anonymized, aggregated signals
- GDPR/CCPA compliant by design

---

## 4. Ontology Collaboration with jjohare

### jjohare's GMC-O Ontology (18KB full)
- Repository: https://github.com/jjohare/hackathon-tv5
- File: `semantic-recommender/design/ontology/expanded-media-ontology.ttl`

### Key Classes
- **Media**: CreativeWork, Film, TVSeries, TVEpisode
- **Psychographic States**: SeekingComfort, SeekingChallenge, Nostalgic, Adventurous
- **Taste Clusters**: Mainstream, ArtHouse, GenreSpecialist
- **Context**: TimeOfDay (EarlyMorning, LateNight), SocialSetting (DateNight, FamilyGathering, SoloViewing)
- **Tolerances**: Violence, Complexity, Subtitles, SlowPacing

### Our Edge Subset (3KB compressed)
We deploy a compressed subset with only edge-relevant classes for WASM deployment.

### Collaboration Model
| jjohare Provides | We Provide |
|------------------|------------|
| Full GMC-O ontology (18KB) | Edge-optimized subset (3KB) |
| GPU inference (32K QPS) | On-device inference (61µs) |
| Cloud-scale reasoning | Edge-scale personalization |
| Multi-modal embeddings | Real-time activity signals |

**Key Point**: This is a collaborative hackathon, not competitive.

---

## 5. Entity Resolution Clarified

In our context, entity resolution serves TWO purposes:

1. **Media Entity Resolution**: Linking same movie/show across TMDB↔IMDb↔Wikidata using Wikidata QIDs
2. **User Entity Resolution**: Linking same user's activity across sessions/devices to build continuous profile

---

## 6. New Agent Swarm

### Old Agents (metadata quality focus)
- Scout, Matcher, Enricher, Validator

### New Agents (activity + ontology focus)
- **CatalogScout**: JustWatch, TMDB sync
- **Enricher**: Embeddings, ontology tagging
- **ProfileBuilder**: User clusters, preference vectors
- **Analytics**: Trend analysis, insights
- **OntologyReasoner**: OWL/SWRL inference

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| On-Device Search (WASM) | 61µs | RuVector HNSW k=10 |
| Edge Search (Cloudflare) | <30ms | 31ms median validated |
| Activity Tracking Overhead | <1ms | Async, non-blocking |
| Profile Update | <10ms | IndexedDB |
| Memory Budget (Edge) | <128MB | 85MB vectors + 30MB runtime |
| Edge Titles | 10K hot | PQ8 quantization |
| Central Titles | 400K+ | Full catalog |

---

## 8. New Demo Strategy

### The Hook
> "We're the only team that brings AI to YOUR device. Search in 61 microseconds. Learn from YOUR behavior. Keep YOUR data private. Work offline."

### Demo Flow
1. **Speed Test**: 61µs local vs 200ms cloud (10,000x faster)
2. **Activity Tracking**: Browse content → watch profile evolve
3. **Psychographic State**: System infers "seeking comfort" from late-night browsing
4. **Privacy Dashboard**: All data stored locally
5. **Offline Mode**: Disconnect WiFi → search still works
6. **Collaborative**: Show how we complement jjohare's GPU inference

---

## 9. Documents Requiring Alignment

### Priority 1: Heavy Rewrites
| Document | Location | Changes |
|----------|----------|---------|
| `executive-summary.md` | `/docs/research/` | Complete rewrite - new differentiator, demo, tech stack |
| `architecture-final.md` | `/docs/research/` | Add Tier 0, activity monitoring, new agents, ontology |

### Priority 2: Medium Updates
| Document | Location | Changes |
|----------|----------|---------|
| `edge-computing-architecture.md` | `/docs/research/` | Move WASM to hackathon, add device tier, activity sync |

### Priority 3: Light Updates
| Document | Location | Changes |
|----------|----------|---------|
| `hackathon-alignment-report.md` | `/docs/research/` | Update coverage %, add new requirements |

### Already Aligned
| Document | Location | Status |
|----------|----------|--------|
| `README.md` | `/workspaces/tv5-fork/` | ✅ Updated |
| `edge-user-activity-architecture.md` | `/docs/research/` | ✅ New, authoritative |

---

## 10. Key Files Reference

### Fork Repository
- URL: https://github.com/binto-labs/hackathon-tv5-AFNZ
- Local: `/workspaces/tv5-fork/`

### Research Documents
- Location: `/workspaces/research/docs/research/`
- Also copied to: `/workspaces/tv5-fork/docs/research/`

### New Authoritative Document
- `/workspaces/research/docs/research/edge-user-activity-architecture.md`
- This is the SOURCE OF TRUTH for the new architecture

---

## 11. Technical Stack (Final)

### Edge Layer (User Device)
- **Vector Engine**: RuVector WASM (61µs search)
- **Index**: HNSW with PQ8 quantization
- **Storage**: IndexedDB for profiles + activity
- **Embeddings**: all-MiniLM-L6-v2 (384 dimensions)

### CDN Layer (Cloudflare)
- Workers + Vectorize (330+ PoPs)
- Workers KV for availability cache
- Activity aggregation with privacy transform

### Central Layer
- AgentDB v1.6.0 (Qdrant fallback)
- PostgreSQL + Redis
- jjohare's GMC-O ontology engine

---

## 12. What NOT to Change

These concepts remain from original research:
- Wikidata QID as canonical identifier
- ARW compliance (llms.txt, JSON-LD, MCP)
- TMDB as primary data source
- JustWatch for streaming availability
- Claude Flow for agent orchestration
- Schema.org compliance

---

## 13. Alignment Instructions

When updating documents:

1. **Replace old tier numbering** with new (Tier 0/1/2)
2. **Add user activity monitoring** section to architecture docs
3. **Replace old agent swarm** (Scout/Matcher/Enricher/Validator) with new
4. **Add jjohare collaboration** and GMC-O ontology references
5. **Update demo strategy** to edge/privacy/activity focus
6. **Update performance targets** to include device-level metrics
7. **Add privacy architecture** section
8. **Reference edge-user-activity-architecture.md** as authoritative source

---

## 14. After Document Alignment

Next step is **High-Level Design (HLD)**:
- System components with interfaces
- Data models (concrete schemas)
- API contracts
- Sequence diagrams
- Deployment architecture
- Hackathon MVP scope

---

## 15. Key URLs

- Hackathon: https://agentics.org/hackathon
- Our Fork: https://github.com/binto-labs/hackathon-tv5-AFNZ
- jjohare: https://github.com/jjohare/hackathon-tv5
- RuVector: https://github.com/ruvnet/ruvector
- GMC-O Ontology: https://github.com/jjohare/hackathon-tv5/blob/main/semantic-recommender/design/ontology/expanded-media-ontology.ttl

---

**END OF CONTEXT REHYDRATION DOCUMENT**

To continue: Read this document, then proceed with document alignment starting with `executive-summary.md`.
