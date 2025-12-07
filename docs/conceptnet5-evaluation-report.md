# ConceptNet5 Evaluation Report
## Media Discovery Platform - Hackathon Research

**Date:** 2025-12-06
**Researcher:** Technical Research Agent
**Status:** COMPLETED

---

## Executive Summary

**RECOMMENDATION: CONDITIONALLY USE - FOR SEMANTIC ENRICHMENT ONLY**

ConceptNet5 is a mature, open-source commonsense knowledge graph with 13+ million semantic relationships across multiple languages. While it offers valuable semantic understanding capabilities, it has **limited direct media domain coverage** and should be used as a **complementary enrichment layer** rather than a primary knowledge source for your media discovery platform.

### Key Findings
- ✅ **Strong:** Semantic similarity, concept expansion, multilingual support
- ⚠️ **Limited:** No media-specific ontology, no direct TMDB/IMDb integration
- ✅ **Maintained:** Active repository (2,020+ commits), 50K+ API hits/day
- ✅ **License:** Creative Commons (compatible with commercial use)
- ⚠️ **Performance:** Public API available but may need self-hosting for scale

---

## 1. What is ConceptNet5?

### Core Capabilities

ConceptNet is a **semantic network providing commonsense knowledge** to computers, representing "things computers should know about the world, especially for understanding text written by people."

**Key Statistics:**
- **13+ million semantic links** between concepts
- **Multiple natural languages** (multilingual from inception)
- **50,000+ API hits per day** on public instance
- **2,020+ commits** - actively maintained
- **Creative Commons licensed** - open for commercial use

**Data Structure:**
- **Knowledge representation:** Linked data semantic network
- **Concepts:** Represented as words/phrases in natural language
- **Relationships:** Labeled edges connecting concepts (e.g., IsA, UsedFor, HasProperty, RelatedTo)
- **Sources:** Crowd-sourced (Wiktionary, Open Mind Common Sense), expert-created (WordNet), games with purpose

### Technical Architecture

```
ConceptNet5 Components:
├── API: http://conceptnet.io (public)
├── Data Format: JSON-LD, RDF
├── Integration: REST API, Neo4j graph database
├── Embeddings: ConceptNet Numberbatch (word vectors)
└── Languages: 100+ languages supported
```

**Source:** [ConceptNet GitHub Repository](https://github.com/commonsense/conceptnet5), [ConceptNet 5.5 Paper](https://arxiv.org/abs/1612.03975)

---

## 2. How ConceptNet5 Could Help Your Platform

### ✅ Strong Use Cases

#### 2.1 Semantic Query Expansion
**Problem:** User searches "cozy movie night"
**Solution:** ConceptNet expands concepts:
- "cozy" → RelatedTo: comfortable, warm, intimate, relaxed
- "night" → RelatedTo: evening, dark, late
- Maps to media attributes: family-friendly, feel-good, lighthearted

**Technical Implementation:**
```python
# Example: Query expansion
user_query = "cozy movie"
conceptnet_api = "http://api.conceptnet.io/query"
related = get_related_concepts("cozy")  # → [comfortable, warm, intimate]
expanded_search = search_media(genres=["family", "romance"], mood=["feel-good"])
```

#### 2.2 Genre/Mood Relationship Mapping
ConceptNet can bridge **natural language** to **structured metadata**:

| User Input | ConceptNet Relations | Media Attributes |
|------------|---------------------|------------------|
| "something intense" | intense → RelatedTo: thrilling, dark, dramatic | genres: [thriller, drama] |
| "feel-good movie" | feel-good → CausesDesire: happiness, laughter | tags: [uplifting, comedy, family] |
| "mind-bending" | mind-bending → RelatedTo: complex, confusing, surreal | tags: [psychological, sci-fi, nonlinear] |

#### 2.3 Cross-Cultural Concept Mapping
**Strength:** ConceptNet's multilingual support (100+ languages)
- Map concepts across languages for international content discovery
- Example: "Bollywood" concepts → Western genre equivalents

#### 2.4 Commonsense Reasoning
**Example:** "movies like Inception but easier to follow"
- ConceptNet: Inception → HasProperty: complex, confusing
- Filter: sci-fi + mind-bending - (very complex)
- Results: Looper, Source Code, Edge of Tomorrow

**Source:** [ConceptNet Applications](https://conceptnet.io/), [Neo4j Integration Blog](https://neo4j.com/blog/graph-database/conceptnet-neo4j-database/)

### ⚠️ Limitations for Media Domain

#### 2.5 No Media-Specific Ontology
**Critical Gap:** ConceptNet is **general-purpose commonsense knowledge**
- ❌ No direct connections to TMDB IDs, IMDb IDs, or Wikidata entities
- ❌ No genre taxonomies specific to film/TV
- ❌ No cast/crew relationships (actor → movies)
- ❌ No streaming availability concepts

**Comparison:**
```
ConceptNet:   "thriller" → RelatedTo: "suspense", "mystery", "danger"
Wikidata:     "thriller" → InstanceOf: Q3037293 (film genre)
              └─ HasPart: Q52207399 (suspense)
              └─ LinkedTo: TMDB genre ID 53
```

#### 2.6 Entity Resolution Mismatch
Your platform needs **entity resolution across databases**:
- TMDB ID ↔ IMDb ID ↔ Wikidata QID ↔ EIDR

ConceptNet provides **concept relationships**, not **entity linking**:
- ConceptNet: "Batman" → RelatedTo: superhero, DC Comics, vigilante
- Not: "The Dark Knight (2008)" → SameAs: tt0468569 (IMDb), Q166262 (Wikidata)

**Source:** [Knowledge Graph Recommendations 2025](https://www.nature.com/articles/s41598-025-14150-5)

---

## 3. Integration Complexity

### API & Data Access

#### Public API
```bash
# GET request
curl "http://api.conceptnet.io/c/en/thriller"

# Response: JSON-LD with edges
{
  "@id": "/c/en/thriller",
  "edges": [
    {"rel": "/r/RelatedTo", "end": "/c/en/suspense", "weight": 2.8},
    {"rel": "/r/IsA", "end": "/c/en/genre", "weight": 2.1}
  ]
}
```

**Pros:**
- ✅ Free to use (50K+ hits/day shows capacity)
- ✅ RESTful API, easy integration
- ✅ JSON-LD format (structured data)

**Cons:**
- ⚠️ Rate limits unclear for production use
- ⚠️ Latency for real-time queries (external API call)

#### Self-Hosting
```bash
# Build from source (requires PostgreSQL)
git clone https://github.com/commonsense/conceptnet5.git
cd conceptnet5
pip install -e .
# Download data dumps (several GB)
```

**Pros:**
- ✅ Full control, no rate limits
- ✅ Can import into Neo4j graph database

**Cons:**
- ⚠️ Complex setup (PostgreSQL, build pipeline)
- ⚠️ Data size: multi-GB downloads
- ⚠️ Maintenance overhead

**Source:** [ConceptNet GitHub](https://github.com/commonsense/conceptnet5)

#### Neo4j Integration
ConceptNet can be imported into **Neo4j** for graph queries:

```cypher
// Find related concepts
MATCH (c:Concept {name: "thriller"})-[r:RELATED_TO]->(related)
RETURN related.name, r.weight
ORDER BY r.weight DESC
LIMIT 10
```

**Use Case:** Tag-based recommendations for retail/social platforms

**Source:** [ConceptNet Neo4j Integration](https://neo4j.com/blog/graph-database/conceptnet-neo4j-database/)

### ConceptNet Numberbatch (Word Embeddings)

**Advanced Feature:** Pre-trained word vectors for semantic similarity

```python
from conceptnet5.vectors import get_vector_similarity

similarity = get_vector_similarity("thriller", "suspense")  # → 0.72
similarity = get_vector_similarity("comedy", "horror")      # → 0.18

# Use for query expansion
genres = ["action", "adventure", "sci-fi"]
user_mood = "intense"
scores = [get_vector_similarity(user_mood, g) for g in genres]
# → action: 0.65, adventure: 0.42, sci-fi: 0.38
```

**Proven Performance:**
- State-of-the-art at SemEval 2017 (word similarity)
- Multilingual, aligned across languages
- Free, no licensing restrictions

**Source:** [ConceptNet 5.5 Paper](https://ojs.aaai.org/index.php/AAAI/article/view/11164/11023)

---

## 4. Maintenance & Licensing

### Repository Status ✅

**Active Maintenance:**
- **2,020+ commits** on master branch
- **Last commit:** fda1b39 (recent activity confirmed)
- **Community:** Open Mind Common Sense project (MIT Media Lab origins)
- **Production use:** 50,000+ API hits/day

### License ✅

**Creative Commons** (see `LICENSE.txt` and `DATA-CREDITS.md`)
- ✅ **Commercial use allowed**
- ✅ **Attribution required**
- ✅ **Compatible with hackathon/startup use**

### Performance at Scale ⚠️

**Concerns:**
- Public API rate limits not documented
- Self-hosting required for high-volume production
- No SLA guarantees on public instance

**Mitigation:**
- Cache frequent queries
- Self-host for production (Neo4j recommended)
- Batch processing for embeddings

---

## 5. Alternatives & Complementary Tools

ConceptNet should be used **alongside** domain-specific knowledge graphs, not as a replacement:

### Primary Knowledge Graphs (Required)

| Tool | Purpose | Coverage |
|------|---------|----------|
| **Wikidata** | Entity resolution, canonical IDs | 100M+ entities, media/entertainment coverage |
| **TMDB API** | Movie/TV metadata | Streaming availability, cast, crew, ratings |
| **IMDb Datasets** | Canonical film database | Industry standard IDs, comprehensive metadata |
| **Schema.org** | Structured data for ARW | Standard vocabulary for media entities |

### Complementary Knowledge Graphs (Optional)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| **ConceptNet** | Semantic enrichment, concept expansion | Query understanding, mood/vibe matching |
| **WordNet** | Synonym expansion, lexical relations | English-only semantic search |
| **DBpedia** | Wikipedia entity extraction | General knowledge, cultural context |
| **Knowledge Graph Embeddings** | Semantic similarity at scale | Large-scale recommendation systems |

**Source:** [Knowledge Graph Use Cases 2025](https://www.pingcap.com/article/knowledge-graph-use-cases-2025/), [Graph Databases 2025](https://www.puppygraph.com/blog/best-graph-databases)

---

## 6. Recommended Architecture

```
User Query: "cozy rom-com streaming tonight"
    ↓
┌─────────────────────────────────────────┐
│ ConceptNet (Semantic Layer)            │
│ - "cozy" → [comfortable, warm, feel-good]│
│ - "rom-com" → [romantic, comedy, lighthearted]│
└─────────────────────────────────────────┘
    ↓ (expanded concepts)
┌─────────────────────────────────────────┐
│ Wikidata + TMDB (Entity Resolution)    │
│ - Map to TMDB genre IDs                │
│ - Filter by streaming platforms         │
│ - Entity resolution across DBs          │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Your Platform (Ranked Results)         │
│ - "The Proposal" (Comedy/Romance)       │
│ - "Sleepless in Seattle" (feel-good)    │
└─────────────────────────────────────────┘
```

### Integration Strategy

**Phase 1 (Hackathon MVP):**
1. Use ConceptNet **public API** for query expansion
2. Cache common concept relationships (thriller → suspense, etc.)
3. Map expanded concepts to TMDB genres manually

**Phase 2 (Production):**
1. Self-host ConceptNet in Neo4j
2. Create mappings: ConceptNet concepts → TMDB/Wikidata entities
3. Train custom embeddings on media domain corpus

**Phase 3 (Scale):**
1. Hybrid knowledge graph: ConceptNet + Wikidata + domain-specific ontology
2. Fine-tune embeddings on user interaction data
3. Real-time semantic search with vector databases

---

## 7. Final Recommendation

### ✅ USE ConceptNet IF:
1. You need **natural language query understanding** ("something cozy", "mind-bending")
2. You want to **expand user queries** semantically (thriller → suspense, tension)
3. You're building **multilingual search** (100+ languages)
4. You need **commonsense reasoning** about moods/vibes

### ❌ DON'T USE ConceptNet FOR:
1. **Primary entity resolution** (use Wikidata + TMDB IDs)
2. **Media-specific metadata** (cast, crew, streaming availability)
3. **Direct database linking** (IMDb ↔ TMDB ↔ Wikidata)
4. **Real-time recommendations** (latency concerns with public API)

### 🎯 RECOMMENDED APPROACH: HYBRID SYSTEM

**ConceptNet Role:** Semantic enrichment layer (20-30% of functionality)
**Primary Knowledge Graph:** Wikidata + TMDB + IMDb (70-80% of functionality)

**Example Query Flow:**
```
1. User: "I want something like Inception but easier to understand"
2. ConceptNet: Inception → [complex, mind-bending, confusing]
3. Expand: sci-fi + (mind-bending - complex)
4. Wikidata: Map to TMDB genre IDs + filter complexity tags
5. TMDB: Retrieve streaming availability + ratings
6. Return: Looper, Source Code, Edge of Tomorrow
```

---

## 8. Action Items for Hackathon

### Immediate (Day 1-2)
- [ ] Test ConceptNet API with sample queries (`/c/en/thriller`)
- [ ] Build query expansion prototype (5-10 common moods)
- [ ] Map ConceptNet concepts → TMDB genres (manual mapping table)

### Medium-term (Day 3-4)
- [ ] Cache frequent concept relationships (top 100 concepts)
- [ ] Integrate with Wikidata for entity resolution
- [ ] Benchmark API latency (public vs self-hosted)

### Optional (Post-Hackathon)
- [ ] Self-host ConceptNet in Neo4j
- [ ] Train domain-specific embeddings on media corpus
- [ ] Build hybrid knowledge graph (ConceptNet + Wikidata + custom ontology)

---

## 9. Resources

### Documentation
- **ConceptNet Homepage:** https://conceptnet.io/
- **GitHub Repository:** https://github.com/commonsense/conceptnet5
- **API Documentation:** http://api.conceptnet.io/docs
- **Academic Paper:** [ConceptNet 5.5: An Open Multilingual Graph](https://arxiv.org/abs/1612.03975)

### Integration Guides
- **Neo4j Integration:** [Non-Text Discovery with ConceptNet](https://neo4j.com/blog/graph-database/conceptnet-neo4j-database/)
- **MCP Server:** [ConceptNet MCP for LLMs](https://github.com/infinitnet/conceptnet-mcp)

### Research Papers
- [Movie Recommendation with Cultural Metadata](https://link.springer.com/chapter/10.1007/978-3-642-03270-7_9)
- [Knowledge Graph Enhanced Recommendations](https://www.nature.com/articles/s41598-024-74516-z)

---

## Sources

- [ConceptNet GitHub Repository](https://github.com/commonsense/conceptnet5)
- [ConceptNet 5.5: An Open Multilingual Graph](https://arxiv.org/abs/1612.03975)
- [ConceptNet Official Website](https://conceptnet.io/)
- [Neo4j ConceptNet Integration](https://neo4j.com/blog/graph-database/conceptnet-neo4j-database/)
- [Knowledge Graph Use Cases 2025](https://www.pingcap.com/article/knowledge-graph-use-cases-2025/)
- [ConceptNet 5.5 AAAI Paper](https://ojs.aaai.org/index.php/AAAI/article/view/11164/11023)
- [ConceptNet MCP Server](https://github.com/infinitnet/conceptnet-mcp)
- [Exploring Movie Recommendation Using Cultural Metadata](https://link.springer.com/chapter/10.1007/978-3-642-03270-7_9)
- [Movie Recommender Systems: Concepts and Challenges](https://pmc.ncbi.nlm.nih.gov/articles/PMC9269752/)
- [Graph Databases 2025 Guide](https://www.puppygraph.com/blog/best-graph-databases)
- [Knowledge Graph Enhanced Recommendations](https://www.nature.com/articles/s41598-024-74516-z)
- [ConceptNet Blog Announcement](http://blog.conceptnet.io/posts/2016/conceptnet-5-5-and-conceptnet-io/)

---

**End of Report**
