---
status: keep
phase: complete
type: spec
version: 1.0
last-updated: 2025-12-06
title: Entity Resolution Specification - TV5 Media Gateway
author: C2-ER-SPEC
---

# Entity Resolution Specification - TV5 Media Gateway

## Executive Summary

This specification defines the entity resolution system for the TV5 Media Gateway, which must match media titles across TMDB, IMDb, and Wikidata with 92%+ precision and 88%+ recall at sub-500ms latency. The system employs a three-tier matching architecture that processes 400,000+ titles using deterministic ID matching (92% coverage), probabilistic fuzzy matching with Splink (7% coverage), and semantic vector similarity (1% edge cases). Wikidata QIDs serve as canonical identifiers, with a fallback hierarchy to TMDB/IMDb IDs when QIDs are unavailable.

**Key Performance Targets:**
- Overall Precision: 92%+ (weighted across tiers)
- Overall Recall: 88%+ (match success rate)
- Latency: <500ms p95 for Tiers 1-2, <2s for Tier 3
- Coverage: 95%+ of queries resolved (any tier)
- Throughput: 100 req/sec sustained (with caching)

## 1. Overview

### 1.1 Purpose

The entity resolution system solves the critical problem of identifying when multiple records across different data sources (TMDB, IMDb, Wikidata) refer to the same real-world media entity. This enables:

1. **Unified discovery** - Search across all sources with single canonical ID
2. **Deduplication** - Prevent duplicate titles in recommendations
3. **Metadata enrichment** - Aggregate data from multiple authoritative sources
4. **Streaming availability** - Link canonical entities to JustWatch/platform data

### 1.2 Scope

**In Scope:**
- Movies and TV series entity resolution
- External ID mapping (TMDB ↔ IMDb ↔ Wikidata)
- Title, year, and basic metadata matching
- Confidence scoring and threshold management
- Unresolved entity queue for manual review

**Out of Scope (Post-Hackathon):**
- Actor/director/crew entity resolution
- Episode-level matching
- Cross-lingual semantic matching (English only for MVP)
- Real-time Wikidata contribution workflow
- Machine learning model training and optimization

### 1.3 Key Assumptions

1. **Wikidata Coverage Gap:** Only 7-10% of TMDB titles have Wikidata QIDs mapped[^1]
2. **SPARQL Timeout Risk:** Wikidata SPARQL endpoint has 60-second query timeout[^2]
3. **TMDB as Primary Source:** TMDB API provides IMDb IDs for most titles[^3]
4. **No Training Data:** System must work without labeled match/non-match pairs
5. **Blocking Required:** Naive comparison of 400K titles = 80 billion pairs (infeasible)

### 1.4 Design Principles

1. **Precision over Recall** - Avoid false positives (wrong matches frustrate users more than missing matches)
2. **Tier-based Fallback** - Start with fast deterministic methods, escalate to slower semantic methods only when needed
3. **Caching First** - Mitigate SPARQL timeout risk with local PostgreSQL cache
4. **Transparent Confidence** - Every match includes confidence score for debugging and validation
5. **Human-in-Loop Ready** - Medium-confidence matches (0.75-0.84) queue for review

---

## 2. Three-Tier Matching Pipeline

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENTITY RESOLUTION REQUEST                     │
│  Input: {tmdb_id?, imdb_id?, title, year, type}                 │
└───────────────────────────┬─────────────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────────────┐
│ TIER 1: DETERMINISTIC ID MATCHING (92% coverage)                  │
│ ----------------------------------------------------------------  │
│ Check local cache (PostgreSQL): tmdb_id OR imdb_id → QID          │
│   ├─ CACHE HIT → Return canonical entity (98%+ confidence)        │
│   └─ CACHE MISS → Query Wikidata SPARQL (P345/P4947 properties)  │
│       ├─ FOUND → Cache + Return (98%+ confidence)                 │
│       └─ NOT FOUND → Proceed to Tier 2                            │
│                                                                   │
│ Performance: <100ms (cached), <500ms (SPARQL)                     │
│ Precision: 98%+ | Recall: 92%                                     │
└───────────────────────────┬───────────────────────────────────────┘
                            ↓ (7-8% of queries)
┌───────────────────────────────────────────────────────────────────┐
│ TIER 2: PROBABILISTIC FUZZY MATCHING (7% coverage)                │
│ ----------------------------------------------------------------  │
│ Splink (Fellegi-Sunter model) with DuckDB backend                 │
│   1. Normalize title (lowercase, remove punctuation)              │
│   2. Generate blocking keys (first 3 chars + year + type)         │
│   3. Retrieve candidate block from local DB                       │
│   4. Splink probabilistic matching:                               │
│      - Jaro-Winkler similarity on title                           │
│      - Exact match on year                                        │
│      - N-gram token comparison                                    │
│   5. Calculate match probability (Fellegi-Sunter formula)         │
│   6. Apply threshold:                                             │
│      - ≥0.85 → Auto-match (cache + return)                        │
│      - 0.75-0.84 → Review queue                                   │
│      - <0.75 → Proceed to Tier 3                                  │
│                                                                   │
│ Performance: 1-5s (with blocking)                                 │
│ Precision: 90%+ | Recall: 85%                                     │
└───────────────────────────┬───────────────────────────────────────┘
                            ↓ (1% of queries)
┌───────────────────────────────────────────────────────────────────┐
│ TIER 3: SEMANTIC VECTOR SIMILARITY (<1% coverage)                 │
│ ----------------------------------------------------------------  │
│ Sentence Transformers + DuckDB HNSW index                          │
│   1. Generate embedding: all-MiniLM-L6-v2 (384 dim)               │
│      - Input: "{title} {year} {type}"                             │
│   2. DuckDB vector search:                                        │
│      - HNSW index query (k=10 nearest neighbors)                  │
│      - Cosine similarity calculation                              │
│      - Metadata filters: type, year±2, language                   │
│   3. Apply threshold:                                             │
│      - ≥0.85 → High confidence (review queue)                     │
│      - 0.70-0.84 → Medium confidence (review queue)               │
│      - <0.70 → Unresolved (create pending task)                   │
│                                                                   │
│ Performance: 100-500ms                                            │
│ Precision: 75%+ | Recall: 70%                                     │
└───────────────────────────┬───────────────────────────────────────┘
                            ↓ (<0.5% of queries)
┌───────────────────────────────────────────────────────────────────┐
│ UNRESOLVED ENTITY QUEUE                                           │
│ ----------------------------------------------------------------  │
│ Store in "pending_resolution" table:                              │
│   - Source data (TMDB/IMDb IDs, title, year, type)               │
│   - Best candidate matches with confidence scores                │
│   - Flagged for manual review or post-hackathon contribution     │
└───────────────────────────────────────────────────────────────────┘
```

### 2.2 Tier Priority and Fallback Logic

**Tier Selection Logic:**

```python
def resolve_entity(tmdb_id=None, imdb_id=None, title=None, year=None, entity_type=None):
    """
    Master entity resolution function.
    Returns: {
        'canonical_id': str,        # Wikidata QID or fallback ID
        'external_ids': dict,       # All known IDs
        'confidence': float,        # 0.0-1.0
        'tier_used': str,          # '1-exact', '2-fuzzy', '3-semantic'
        'metadata': dict           # Source metadata
    }
    """
    # TIER 1: External ID Lookup
    if tmdb_id or imdb_id:
        result = tier1_exact_match(tmdb_id, imdb_id)
        if result:
            return result  # 98%+ confidence

    # TIER 2: Fuzzy Matching (requires title)
    if title and year:
        result = tier2_fuzzy_match(title, year, entity_type)
        if result and result['confidence'] >= 0.85:
            return result  # Auto-match
        if result and result['confidence'] >= 0.75:
            return enqueue_for_review(result, reason="medium_confidence")

    # TIER 3: Semantic Vector Search (last resort)
    if title:
        result = tier3_semantic_match(title, year, entity_type)
        if result and result['confidence'] >= 0.70:
            return enqueue_for_review(result, reason="semantic_match")

    # UNRESOLVED
    return create_pending_resolution_task(
        tmdb_id=tmdb_id,
        imdb_id=imdb_id,
        title=title,
        year=year,
        entity_type=entity_type,
        candidates=[tier2_result, tier3_result] if available
    )
```

**Coverage Distribution (Expected):**
- Tier 1 (Exact ID): 92% of queries (370K out of 400K titles)
- Tier 2 (Fuzzy): 7% of queries (28K titles)
- Tier 3 (Semantic): 1% of queries (4K titles)
- Unresolved: <0.5% (2K titles → manual review queue)

---

## 3. Tier 1: Deterministic ID Matching

### 3.1 External ID Hierarchy

**Wikidata as Canonical Identifier:**
- **Primary:** Wikidata QID (Q-number, e.g., Q174284 = "The Shawshank Redemption")
- **Rationale:** Stable, language-agnostic, versioned, community-maintained, neutral (not company-owned)

**Fallback Hierarchy (when QID unavailable):**
1. TMDB ID (primary metadata source, 400K+ coverage)
2. IMDb ID (high recognition, cross-platform standard)
3. Generated UUID (last resort, temporary until resolved)

**Example Canonical Entity Structure:**
```json
{
  "canonical_id": "Q174284",
  "id_type": "wikidata_qid",
  "external_ids": {
    "wikidata": "Q174284",
    "tmdb_movie": "278",
    "imdb": "tt0111161",
    "tvdb": null,
    "justwatch": "tm/movie/278"
  },
  "resolution_metadata": {
    "tier_used": "1-exact",
    "confidence": 0.98,
    "source": "wikidata_sparql",
    "last_updated": "2025-12-06T10:30:00Z"
  }
}
```

### 3.2 SPARQL Query Patterns

**Pattern 1: IMDb ID → Wikidata QID**

```sparql
SELECT ?item ?itemLabel ?tmdbId ?tvdbId WHERE {
  ?item wdt:P345 "tt0111161" .  # IMDb ID (P345)
  OPTIONAL { ?item wdt:P4947 ?tmdbId . }  # TMDB movie ID (P4947)
  OPTIONAL { ?item wdt:P4983 ?tmdbTvId . }  # TMDB TV series ID (P4983)
  OPTIONAL { ?item wdt:P4944 ?tvdbId . }  # TVDB ID (P4944)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en" . }
}
```

**Performance:** <100ms for simple lookups[^2]

**Pattern 2: TMDB ID → Full External ID Set**

```sparql
SELECT ?item ?itemLabel ?imdbId ?tvdbId ?rottenTomatoesId WHERE {
  {
    ?item wdt:P4947 "278" .  # TMDB movie ID
  } UNION {
    ?item wdt:P4983 "278" .  # TMDB TV series ID
  }
  OPTIONAL { ?item wdt:P345 ?imdbId . }
  OPTIONAL { ?item wdt:P4944 ?tvdbId . }
  OPTIONAL { ?item wdt:P1267 ?rottenTomatoesId . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en" . }
}
```

**Performance:** 100-500ms for property queries[^2]

**Pattern 3: Batch ID Lookup (Cache Refresh)**

```sparql
SELECT ?item ?imdbId ?tmdbMovieId ?tmdbTvId WHERE {
  VALUES ?imdbId { "tt0111161" "tt0068646" "tt0071562" ... }  # Batch of 100
  ?item wdt:P345 ?imdbId .
  OPTIONAL { ?item wdt:P4947 ?tmdbMovieId . }
  OPTIONAL { ?item wdt:P4983 ?tmdbTvId . }
}
LIMIT 100
```

**Performance:** 500ms-5s for 100 IDs
**Rationale:** Amortize SPARQL overhead across multiple lookups

### 3.3 Local Caching Strategy

**Cache Structure (PostgreSQL):**

```sql
CREATE TABLE entity_cache (
    canonical_id VARCHAR(20) PRIMARY KEY,  -- Wikidata QID or fallback ID
    id_type VARCHAR(20) NOT NULL,          -- 'wikidata_qid', 'tmdb_id', 'imdb_id', 'uuid'

    -- External ID indexes
    wikidata_qid VARCHAR(20) UNIQUE,
    tmdb_movie_id VARCHAR(20),
    tmdb_tv_id VARCHAR(20),
    imdb_id VARCHAR(20),
    tvdb_id VARCHAR(20),
    justwatch_id VARCHAR(50),

    -- Resolution metadata
    tier_used VARCHAR(20) NOT NULL,        -- '1-exact', '2-fuzzy', '3-semantic'
    confidence NUMERIC(4,3) NOT NULL,      -- 0.000-1.000
    source VARCHAR(50) NOT NULL,           -- 'wikidata_sparql', 'splink', 'vector_search'

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    last_updated TIMESTAMP DEFAULT NOW(),
    cache_expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '30 days',

    -- Metadata JSON
    metadata JSONB,

    -- Indexes for fast lookups
    CONSTRAINT unique_tmdb_movie UNIQUE NULLS NOT DISTINCT (tmdb_movie_id),
    CONSTRAINT unique_tmdb_tv UNIQUE NULLS NOT DISTINCT (tmdb_tv_id),
    CONSTRAINT unique_imdb UNIQUE NULLS NOT DISTINCT (imdb_id)
);

CREATE INDEX idx_entity_cache_tmdb_movie ON entity_cache(tmdb_movie_id) WHERE tmdb_movie_id IS NOT NULL;
CREATE INDEX idx_entity_cache_tmdb_tv ON entity_cache(tmdb_tv_id) WHERE tmdb_tv_id IS NOT NULL;
CREATE INDEX idx_entity_cache_imdb ON entity_cache(imdb_id) WHERE imdb_id IS NOT NULL;
CREATE INDEX idx_entity_cache_expires ON entity_cache(cache_expires_at);
```

**Cache Population Strategy:**

1. **On-Demand (Real-Time):**
   - Query cache first for every resolution request
   - On cache miss, query Wikidata SPARQL
   - Store result in cache with 30-day TTL
   - Performance: <10ms (cache hit), <500ms (cache miss + SPARQL)

2. **Batch Pre-Population (Offline):**
   - Nightly job: Fetch 100K most-popular TMDB titles
   - Query Wikidata in batches of 100 (5 seconds per batch)
   - Total time: ~1.5 hours for 100K titles
   - Reduces cache miss rate from 93% to <10%

3. **Incremental Updates (Daily):**
   - Subscribe to Wikidata Recent Changes API (optional post-hackathon)
   - Update cache for modified P345/P4947 properties
   - Refresh expiring cache entries (WHERE cache_expires_at < NOW() + INTERVAL '7 days')

**Cache Invalidation:**
- TTL-based: 30 days (aligns with monthly Wikidata contribution cycles)
- Manual: Admin endpoint to purge specific canonical_id
- Bulk: Clear all entries with confidence <0.90 (quarterly cleanup)

### 3.4 Tier 1 Implementation Pseudocode

```python
def tier1_exact_match(tmdb_id=None, imdb_id=None):
    """
    Deterministic external ID matching via Wikidata SPARQL + local cache.

    Returns:
        dict | None: Canonical entity record or None if not found
    """
    # Step 1: Check local cache (PostgreSQL)
    cache_query = """
        SELECT canonical_id, id_type, wikidata_qid, tmdb_movie_id, tmdb_tv_id,
               imdb_id, tvdb_id, justwatch_id, confidence, tier_used, metadata
        FROM entity_cache
        WHERE (tmdb_movie_id = %s OR tmdb_tv_id = %s OR imdb_id = %s)
          AND cache_expires_at > NOW()
        LIMIT 1
    """
    cache_result = db.execute(cache_query, (tmdb_id, tmdb_id, imdb_id))

    if cache_result:
        logger.info(f"Cache HIT: {cache_result['canonical_id']}")
        return format_canonical_entity(cache_result, tier='1-exact')

    # Step 2: Cache miss - Query Wikidata SPARQL
    logger.info(f"Cache MISS: Querying Wikidata for tmdb={tmdb_id}, imdb={imdb_id}")

    sparql_query = construct_sparql_query(tmdb_id, imdb_id)

    try:
        sparql_result = wikidata_sparql_client.query(
            sparql_query,
            timeout=10  # 10-second timeout (well below 60s limit)
        )
    except SPARQLTimeoutError:
        logger.warning(f"SPARQL timeout for tmdb={tmdb_id}, imdb={imdb_id}")
        return None  # Fallback to Tier 2
    except SPARQLError as e:
        logger.error(f"SPARQL error: {e}")
        return None

    if not sparql_result or len(sparql_result['results']['bindings']) == 0:
        logger.info(f"No Wikidata match found for tmdb={tmdb_id}, imdb={imdb_id}")
        return None  # Fallback to Tier 2

    # Step 3: Parse SPARQL response
    binding = sparql_result['results']['bindings'][0]
    wikidata_qid = extract_qid(binding['item']['value'])
    external_ids = {
        'wikidata': wikidata_qid,
        'imdb': binding.get('imdbId', {}).get('value'),
        'tmdb_movie': binding.get('tmdbMovieId', {}).get('value'),
        'tmdb_tv': binding.get('tmdbTvId', {}).get('value'),
        'tvdb': binding.get('tvdbId', {}).get('value')
    }

    # Step 4: Cache result
    cache_insert = """
        INSERT INTO entity_cache (
            canonical_id, id_type, wikidata_qid, tmdb_movie_id, tmdb_tv_id,
            imdb_id, tier_used, confidence, source, metadata
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON CONFLICT (canonical_id) DO UPDATE
        SET last_updated = NOW(), cache_expires_at = NOW() + INTERVAL '30 days'
    """
    db.execute(cache_insert, (
        wikidata_qid,
        'wikidata_qid',
        wikidata_qid,
        external_ids.get('tmdb_movie'),
        external_ids.get('tmdb_tv'),
        external_ids.get('imdb'),
        '1-exact',
        0.98,  # High confidence for external ID matches
        'wikidata_sparql',
        json.dumps({'binding': binding})
    ))

    # Step 5: Return canonical entity
    return {
        'canonical_id': wikidata_qid,
        'id_type': 'wikidata_qid',
        'external_ids': external_ids,
        'confidence': 0.98,
        'tier_used': '1-exact',
        'metadata': {'source': 'wikidata_sparql'}
    }

def construct_sparql_query(tmdb_id, imdb_id):
    """Build SPARQL query for external ID lookup."""
    conditions = []
    if imdb_id:
        conditions.append(f'?item wdt:P345 "{imdb_id}" .')
    if tmdb_id:
        conditions.append(f'{{ ?item wdt:P4947 "{tmdb_id}" . }} UNION {{ ?item wdt:P4983 "{tmdb_id}" . }}')

    where_clause = ' UNION '.join([f'{{ {cond} }}' for cond in conditions])

    return f"""
        SELECT ?item ?itemLabel ?imdbId ?tmdbMovieId ?tmdbTvId ?tvdbId WHERE {{
          {where_clause}
          OPTIONAL {{ ?item wdt:P345 ?imdbId . }}
          OPTIONAL {{ ?item wdt:P4947 ?tmdbMovieId . }}
          OPTIONAL {{ ?item wdt:P4983 ?tmdbTvId . }}
          OPTIONAL {{ ?item wdt:P4944 ?tvdbId . }}
          SERVICE wikibase:label {{ bd:serviceParam wikibase:language "en" . }}
        }}
        LIMIT 1
    """

def extract_qid(wikidata_uri):
    """Extract QID from Wikidata URI."""
    # Example: http://www.wikidata.org/entity/Q174284 → Q174284
    return wikidata_uri.split('/')[-1]
```

### 3.5 Tier 1 Quality Metrics

**Target Metrics:**
- **Precision:** 98%+ (external IDs are highly reliable)
- **Recall:** 92% (based on 7-10% Wikidata coverage gap[^1])
- **Latency:** <10ms (cache hit), <500ms (cache miss + SPARQL)
- **Cache Hit Rate:** 90%+ (after batch pre-population)

**Monitoring:**
```python
tier1_metrics = {
    'total_queries': 10000,
    'cache_hits': 9200,           # 92% cache hit rate
    'cache_misses': 800,           # 8% cache misses
    'sparql_success': 750,         # 93.75% SPARQL success rate
    'sparql_timeout': 30,          # 3.75% timeouts
    'sparql_no_match': 20,         # 2.5% no match found
    'avg_latency_cache_hit': 8,   # milliseconds
    'avg_latency_sparql': 320,    # milliseconds
    'p95_latency': 480,           # milliseconds
    'p99_latency': 850            # milliseconds
}
```

---

## 4. Tier 2: Probabilistic Fuzzy Matching

### 4.1 Splink Configuration

**Splink Overview:**
- Probabilistic record linkage based on Fellegi-Sunter model[^4]
- Unsupervised (no training data required)
- Production-proven at scale (UK government, Australian census)
- DuckDB backend: 1M records in ~1 minute on laptop[^5]

**Model Settings:**

```python
from splink.duckdb.linker import DuckDBLinker
import splink.duckdb.comparison_library as cl
import splink.duckdb.comparison_template_library as ctl

splink_settings = {
    "link_type": "dedupe_only",  # Find duplicates within single dataset

    # Blocking rules (reduce comparison space)
    "blocking_rules_to_generate_predictions": [
        "l.block_l1 = r.block_l1",  # First 3 chars + type
        "l.block_l2 = r.block_l2",  # Soundex + year decade
        "l.year = r.year AND l.entity_type = r.entity_type"
    ],

    # Comparison functions
    "comparisons": [
        # Title comparison (primary signal)
        cl.jaro_winkler_at_thresholds(
            "title_normalized",
            [0.95, 0.85, 0.70],
            term_frequency_adjustments=True  # Penalize common titles
        ),

        # N-gram token comparison
        ctl.n_gram_comparison(
            "title_normalized",
            n=3,
            term_frequency_adjustments=True
        ),

        # Year comparison
        cl.exact_match("year", term_frequency_adjustments=False),

        # Type comparison (movie, series, documentary)
        cl.exact_match("entity_type", term_frequency_adjustments=False),

        # IMDb ID comparison (if available)
        cl.exact_match("imdb_id", term_frequency_adjustments=False),

        # Cast overlap (if metadata available)
        cl.jaccard_at_thresholds("cast_list", [0.7, 0.5, 0.3])
    ],

    # EM training settings
    "em_convergence": 0.01,  # Expectation-Maximization convergence threshold
    "max_iterations": 20,

    # Probability thresholds
    "retain_matching_columns": True,
    "retain_intermediate_calculation_columns": True
}
```

### 4.2 Blocking Strategy

**Multi-Level Blocking Keys:**

```python
def generate_blocking_keys(title, year, entity_type):
    """
    Generate multiple blocking keys to reduce comparison space.

    Naive comparison: 400K titles = 80 billion pairs
    With blocking: ~8 million pairs (99.99% reduction)
    """
    from jellyfish import soundex

    # Normalize title
    title_norm = normalize_title(title)

    # Level 1: Coarse blocking (first 3 chars + type)
    # Expected reduction: 99% (80B → 800M pairs)
    block_l1 = f"{title_norm[:3]}_{entity_type}"

    # Level 2: Medium blocking (soundex + year decade)
    # Expected reduction: 99.9% (80B → 80M pairs)
    soundex_code = soundex(title_norm) if title_norm else 'Z000'
    year_decade = (year // 10) * 10 if year else 0
    block_l2 = f"{soundex_code}_{year_decade}"

    # Level 3: Fine blocking (title length + year)
    # Expected reduction: 99.99% (80B → 8M pairs)
    title_len_bucket = (len(title_norm) // 5) * 5  # Bucket by 5 chars
    block_l3 = f"{title_len_bucket}_{year}"

    return {
        'block_l1': block_l1,
        'block_l2': block_l2,
        'block_l3': block_l3
    }

def normalize_title(title):
    """
    Normalize title for consistent comparison.

    Examples:
        "The Shawshank Redemption" → "shawshank redemption"
        "The Office (US)" → "office us"
        "Matrix, The" → "matrix"
    """
    import re

    if not title:
        return ""

    # Lowercase
    title = title.lower()

    # Remove leading articles (the, a, an)
    title = re.sub(r'^(the|a|an)\s+', '', title)

    # Remove trailing articles (for inverted titles)
    title = re.sub(r',\s+(the|a|an)$', '', title)

    # Remove punctuation except spaces
    title = re.sub(r'[^\w\s]', '', title)

    # Remove extra whitespace
    title = ' '.join(title.split())

    return title
```

**Blocking Key Storage:**

```sql
CREATE TABLE entity_blocks (
    entity_id VARCHAR(50) PRIMARY KEY,

    -- Normalized fields
    title_normalized VARCHAR(500) NOT NULL,
    year INTEGER,
    entity_type VARCHAR(20) NOT NULL,  -- 'movie', 'series', 'documentary'

    -- Blocking keys
    block_l1 VARCHAR(50),  -- First 3 chars + type
    block_l2 VARCHAR(50),  -- Soundex + year decade
    block_l3 VARCHAR(50),  -- Title length + year

    -- External IDs (for Tier 1 fallback check)
    imdb_id VARCHAR(20),
    tmdb_id VARCHAR(20),

    -- Metadata for comparison
    cast_list TEXT[],      -- Top 5 cast members
    director VARCHAR(200),
    plot_summary TEXT,

    -- Indexes for blocking
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_entity_blocks_l1 ON entity_blocks(block_l1);
CREATE INDEX idx_entity_blocks_l2 ON entity_blocks(block_l2);
CREATE INDEX idx_entity_blocks_l3 ON entity_blocks(block_l3);
CREATE INDEX idx_entity_blocks_year_type ON entity_blocks(year, entity_type);
```

### 4.3 Fellegi-Sunter Match Probability

**Formula:**

```
Match Weight (field) = log₂(m / u)

where:
  m = P(agreement | match)     = Data quality (how often matching records agree)
  u = P(agreement | non-match) = Coincidence rate (how often random records agree)

Total Match Weight = Σ(field weights)

Match Probability = (prior_odds × 2^total_weight) / (1 + prior_odds × 2^total_weight)
```

**Field Weight Table:**

| Field | m (match) | u (coincidence) | Weight = log₂(m/u) | Rationale |
|-------|-----------|-----------------|---------------------|-----------|
| **IMDb ID exact** | 0.99 | 0.00001 | +16.6 | Extremely strong (unique identifier) |
| **TMDB ID exact** | 0.99 | 0.0001 | +13.3 | Very strong (unique identifier) |
| **Title exact** | 0.95 | 0.001 | +9.9 | Strong (common titles reduce u) |
| **Title Jaro-Winkler >0.95** | 0.90 | 0.01 | +6.5 | Strong (typos, variants) |
| **Title Jaro-Winkler 0.85-0.95** | 0.70 | 0.05 | +3.8 | Medium (related titles) |
| **Year exact** | 0.90 | 0.05 | +4.2 | Medium (remakes exist) |
| **Director match** | 0.85 | 0.01 | +6.4 | Strong (unique directors) |
| **Cast overlap (3+)** | 0.80 | 0.005 | +7.3 | Strong (ensemble casts) |
| **Type exact** | 0.95 | 0.33 | +1.5 | Weak (only 3 types: movie/series/doc) |

**Example Calculation:**

```python
def calculate_match_probability(field_scores, prior_odds=0.01):
    """
    Fellegi-Sunter match probability calculation.

    Args:
        field_scores: dict of {field: weight}
        prior_odds: Prior probability ratio (default 1%)

    Returns:
        float: Match probability (0.0-1.0)
    """
    # Sum log-likelihood ratios
    total_weight = sum(field_scores.values())

    # Convert to probability using Bayes' theorem
    posterior_odds = prior_odds * (2 ** total_weight)
    probability = posterior_odds / (1 + posterior_odds)

    return probability

# Example 1: Title + Year + Type match
scores_medium = {
    'title_jaro_winkler_0.85': 3.8,
    'year_exact': 4.2,
    'type_exact': 1.5
}
prob_medium = calculate_match_probability(scores_medium)
# Result: 0.71 (71% confidence - review queue)

# Example 2: Title + Year + IMDb ID match
scores_high = {
    'title_exact': 9.9,
    'year_exact': 4.2,
    'imdb_id_exact': 16.6
}
prob_high = calculate_match_probability(scores_high)
# Result: ~0.9999 (virtually certain match)

# Example 3: Title + Year + Director + Cast match
scores_strong = {
    'title_jaro_winkler_0.95': 6.5,
    'year_exact': 4.2,
    'director_match': 6.4,
    'cast_overlap_3': 7.3
}
prob_strong = calculate_match_probability(scores_strong)
# Result: 0.94 (94% confidence - auto-match)
```

### 4.4 Tier 2 Implementation Pseudocode

```python
def tier2_fuzzy_match(title, year=None, entity_type=None):
    """
    Probabilistic fuzzy matching using Splink (Fellegi-Sunter model).

    Returns:
        dict | None: Best match with confidence score, or None if below threshold
    """
    # Step 1: Normalize title
    title_norm = normalize_title(title)

    # Step 2: Generate blocking keys
    blocks = generate_blocking_keys(title_norm, year, entity_type)

    # Step 3: Retrieve candidates from block
    candidate_query = """
        SELECT entity_id, title_normalized, year, entity_type,
               imdb_id, tmdb_id, cast_list, director
        FROM entity_blocks
        WHERE (block_l1 = %s OR block_l2 = %s OR block_l3 = %s)
          AND entity_type = %s
        LIMIT 1000  -- Prevent runaway queries
    """
    candidates = db.execute(candidate_query, (
        blocks['block_l1'],
        blocks['block_l2'],
        blocks['block_l3'],
        entity_type
    ))

    if not candidates or len(candidates) == 0:
        logger.info(f"No candidates found in blocks for title='{title}'")
        return None

    logger.info(f"Retrieved {len(candidates)} candidates from blocks")

    # Step 4: Prepare Splink input
    query_record = {
        'entity_id': 'QUERY',
        'title_normalized': title_norm,
        'year': year,
        'entity_type': entity_type,
        'imdb_id': None,
        'tmdb_id': None,
        'cast_list': [],
        'director': None
    }

    # Combine query + candidates into DataFrame
    import pandas as pd
    df = pd.DataFrame([query_record] + candidates)

    # Step 5: Run Splink matching
    from splink.duckdb.linker import DuckDBLinker

    linker = DuckDBLinker(df, splink_settings, connection=duckdb_conn)

    # Estimate u probabilities (unsupervised)
    linker.estimate_u_using_random_sampling(max_pairs=1e6)

    # Estimate m probabilities using EM algorithm
    linker.estimate_parameters_using_expectation_maximisation(
        blocking_rule="l.block_l1 = r.block_l1"
    )

    # Predict matches
    predictions = linker.predict(threshold_match_probability=0.70)

    # Step 6: Filter results (only matches to QUERY record)
    query_matches = predictions[
        (predictions['entity_id_l'] == 'QUERY') |
        (predictions['entity_id_r'] == 'QUERY')
    ].sort_values('match_probability', ascending=False)

    if len(query_matches) == 0:
        logger.info(f"No matches above 0.70 threshold for title='{title}'")
        return None

    # Step 7: Return best match
    best_match = query_matches.iloc[0]
    matched_entity_id = (
        best_match['entity_id_r'] if best_match['entity_id_l'] == 'QUERY'
        else best_match['entity_id_l']
    )

    # Fetch full entity record
    entity = db.execute(
        "SELECT * FROM entity_blocks WHERE entity_id = %s",
        (matched_entity_id,)
    )[0]

    return {
        'canonical_id': entity['tmdb_id'] or entity['imdb_id'] or matched_entity_id,
        'id_type': 'tmdb_id' if entity['tmdb_id'] else 'imdb_id',
        'external_ids': {
            'tmdb_movie': entity['tmdb_id'],
            'imdb': entity['imdb_id']
        },
        'confidence': float(best_match['match_probability']),
        'tier_used': '2-fuzzy',
        'metadata': {
            'source': 'splink',
            'matched_entity_id': matched_entity_id,
            'matched_title': entity['title_normalized'],
            'comparison_scores': {
                'title_similarity': best_match.get('title_normalized_jaro_winkler'),
                'year_match': best_match.get('year_exact'),
                'type_match': best_match.get('entity_type_exact')
            }
        }
    }
```

### 4.5 Tier 2 Quality Metrics

**Target Metrics:**
- **Precision:** 90%+ (avoid false positives)
- **Recall:** 85% (capture most fuzzy matches)
- **Latency:** 1-5 seconds (with blocking, depends on block size)
- **Block Size:** <1,000 candidates per query (prevent performance degradation)

**Monitoring:**
```python
tier2_metrics = {
    'total_queries': 700,          # 7% of total (Tier 1 fallthrough)
    'candidates_avg': 150,         # Average candidates per block
    'candidates_p95': 450,         # 95th percentile
    'matches_above_0.85': 520,     # 74% auto-match rate
    'matches_0.75_to_0.84': 120,   # 17% review queue
    'matches_below_0.75': 60,      # 9% fallthrough to Tier 3
    'avg_latency': 2300,           # milliseconds
    'p95_latency': 4800,           # milliseconds
    'precision_validated': 0.91,   # From manual review sample
    'recall_estimated': 0.87       # From ground truth sample
}
```

---

## 5. Tier 3: Semantic Vector Similarity

### 5.1 Embedding Model Selection

**Recommended Model: all-MiniLM-L6-v2**

| Property | Value | Rationale |
|----------|-------|-----------|
| **Dimensions** | 384 | 4x less memory than 1536-dim (OpenAI), 2x less than 768-dim models |
| **Parameters** | 22M | Fast CPU inference (5-14k sentences/sec) |
| **Speed** | 5-14k sent/sec | 4-5x faster than 768-dim models[^6] |
| **Quality** | Good | Proven accuracy for retrieval tasks (matches 1536-dim performance)[^7] |
| **Memory** | 1.5GB per 1M vectors | 75% less memory than OpenAI embeddings |
| **License** | Apache 2.0 | Free, open-source, offline-capable |

**Alternative: all-mpnet-base-v2 (768-dim)**
- Use if testing shows need for finer semantic distinctions
- 2x memory (3GB per 1M vectors), 4x slower (1-3k sent/sec)

**Embedding Generation:**

```python
from sentence_transformers import SentenceTransformer

# Load model (once at startup)
model = SentenceTransformer('all-MiniLM-L6-v2')

def generate_title_embedding(title, year=None, entity_type=None):
    """
    Generate 384-dimensional embedding for title matching.

    Input format: "{title} {year} {type}"
    Example: "The Office 2005 series" → [0.12, -0.45, 0.67, ...]
    """
    # Combine fields into single text
    text_parts = [title]
    if year:
        text_parts.append(str(year))
    if entity_type:
        text_parts.append(entity_type)

    input_text = ' '.join(text_parts)

    # Generate embedding
    embedding = model.encode(input_text, normalize_embeddings=True)
    # normalize_embeddings=True ensures cosine similarity is just dot product

    return embedding.tolist()  # Convert numpy array to list for JSON storage
```

### 5.2 DuckDB HNSW Index Configuration

**HNSW (Hierarchical Navigable Small World) Overview:**
- Graph-based algorithm for approximate nearest neighbor search
- O(log n) search complexity (vs O(n) for brute force)
- Accuracy: 95-99% recall at 10x-100x speedup[^8]

**DuckDB VSS Extension Setup:**

```sql
-- Install and load VSS extension
INSTALL vss;
LOAD vss;

-- Create embeddings table
CREATE TABLE media_embeddings (
    entity_id VARCHAR(50) PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    year INTEGER,
    entity_type VARCHAR(20) NOT NULL,

    -- External IDs
    imdb_id VARCHAR(20),
    tmdb_id VARCHAR(20),

    -- 384-dimensional embedding
    embedding FLOAT[384] NOT NULL,

    -- Metadata for filtering
    language VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create HNSW index
CREATE INDEX hnsw_idx ON media_embeddings
USING HNSW (embedding)
WITH (
    metric = 'cosine',        -- Cosine similarity (normalized vectors)
    ef_construction = 200,    -- Build-time accuracy (higher = better quality, slower build)
    M = 16                    -- Graph connectivity (higher = better accuracy, more memory)
);

-- Indexes for metadata filtering
CREATE INDEX idx_media_embeddings_type ON media_embeddings(entity_type);
CREATE INDEX idx_media_embeddings_year ON media_embeddings(year);
CREATE INDEX idx_media_embeddings_lang ON media_embeddings(language);
```

**HNSW Parameter Tuning:**

| Parameter | MVP (100K) | Production (1M) | Large Scale (10M) |
|-----------|------------|------------------|-------------------|
| **M** | 16 | 24 | 32 |
| **ef_construction** | 200 | 400 | 800 |
| **ef_search** | 50 | 100 | 200 |
| **Memory Overhead** | 1.4x | 1.6x | 2.0x |
| **Build Time** | 30s | 10min | 60min |
| **Query Time** | <10ms | <20ms | <50ms |

**Rationale:**
- Higher M = more edges per node = better accuracy, more memory
- Higher ef_construction = more candidates during build = better index quality, slower build
- Higher ef_search = more candidates during search = better accuracy, slower queries

### 5.3 Vector Search Query Patterns

**Pattern 1: Simple k-NN Search**

```sql
-- Find 10 nearest neighbors by cosine similarity
SELECT
    entity_id,
    title,
    year,
    entity_type,
    array_cosine_similarity(embedding, ?) AS similarity
FROM media_embeddings
ORDER BY similarity DESC
LIMIT 10;
```

**Pattern 2: Filtered Vector Search**

```sql
-- Find nearest neighbors with metadata filters
SELECT
    entity_id,
    title,
    year,
    entity_type,
    array_cosine_similarity(embedding, ?) AS similarity
FROM media_embeddings
WHERE entity_type = 'series'
  AND year BETWEEN 2003 AND 2007
  AND language = 'en'
  AND similarity > 0.70  -- Threshold filter
ORDER BY similarity DESC
LIMIT 10;
```

**Pattern 3: Hybrid Search (Vector + Keyword)**

```sql
-- Combine vector similarity with text matching
WITH vector_results AS (
    SELECT entity_id, title, year,
           array_cosine_similarity(embedding, ?) AS vector_score
    FROM media_embeddings
    WHERE entity_type = 'series'
    ORDER BY vector_score DESC
    LIMIT 50
),
keyword_results AS (
    SELECT entity_id, title, year,
           fts_main_media_embeddings.match_bm25(
               entity_id, 'office'
           ) AS keyword_score
    FROM media_embeddings
    WHERE entity_type = 'series'
    ORDER BY keyword_score DESC
    LIMIT 50
)
SELECT DISTINCT
    COALESCE(v.entity_id, k.entity_id) AS entity_id,
    COALESCE(v.title, k.title) AS title,
    COALESCE(v.year, k.year) AS year,
    COALESCE(v.vector_score, 0.0) * 0.7 +
    COALESCE(k.keyword_score, 0.0) * 0.3 AS combined_score
FROM vector_results v
FULL OUTER JOIN keyword_results k USING (entity_id)
ORDER BY combined_score DESC
LIMIT 10;
```

### 5.4 Similarity Threshold Tuning

**Recommended Thresholds (based on entity resolution research[^9]):**

| Threshold | Interpretation | Use Case | Action |
|-----------|----------------|----------|--------|
| **≥ 0.95** | Near-exact match | "The Office" vs "Office, The" | Auto-match (high confidence) |
| **0.85-0.94** | Strong fuzzy match | "The Office US" vs "The Office" | Review queue (medium confidence) |
| **0.70-0.84** | Moderate similarity | "The Office" vs "The Office: Webisodes" | Review queue (low-medium confidence) |
| **< 0.70** | Weak/unrelated | "The Office" vs "The Wire" | Reject (unresolved) |

**Tuning Process:**

```python
def tune_similarity_threshold(validation_pairs):
    """
    Determine optimal threshold from labeled validation set.

    Args:
        validation_pairs: List of (title1, title2, is_match) tuples

    Returns:
        dict: Optimal thresholds and metrics
    """
    import numpy as np
    from sklearn.metrics import precision_recall_curve, f1_score

    # Generate embeddings for validation pairs
    embeddings1 = [generate_title_embedding(t1) for t1, _, _ in validation_pairs]
    embeddings2 = [generate_title_embedding(t2) for _, t2, _ in validation_pairs]

    # Calculate cosine similarities
    similarities = [
        np.dot(e1, e2) for e1, e2 in zip(embeddings1, embeddings2)
    ]

    # Extract ground truth labels
    labels = [is_match for _, _, is_match in validation_pairs]

    # Calculate precision-recall curve
    precisions, recalls, thresholds = precision_recall_curve(labels, similarities)

    # Find threshold maximizing F1 score
    f1_scores = 2 * (precisions * recalls) / (precisions + recalls + 1e-10)
    best_idx = np.argmax(f1_scores)

    return {
        'optimal_threshold': thresholds[best_idx],
        'precision': precisions[best_idx],
        'recall': recalls[best_idx],
        'f1_score': f1_scores[best_idx],
        'threshold_curve': list(zip(thresholds, precisions, recalls, f1_scores))
    }
```

### 5.5 Tier 3 Implementation Pseudocode

```python
def tier3_semantic_match(title, year=None, entity_type=None):
    """
    Semantic vector similarity search using Sentence Transformers + DuckDB HNSW.

    Returns:
        dict | None: Best match with confidence score, or None if below threshold
    """
    # Step 1: Generate query embedding
    query_embedding = generate_title_embedding(title, year, entity_type)

    # Step 2: DuckDB vector search with filters
    search_query = """
        SELECT
            entity_id,
            title,
            year,
            entity_type,
            imdb_id,
            tmdb_id,
            array_cosine_similarity(embedding, ?::FLOAT[384]) AS similarity
        FROM media_embeddings
        WHERE entity_type = ?
          AND (year IS NULL OR year BETWEEN ? AND ?)
          AND similarity > 0.70
        ORDER BY similarity DESC
        LIMIT 10
    """

    year_min = (year - 2) if year else 1900
    year_max = (year + 2) if year else 2030

    results = duckdb_conn.execute(search_query, (
        query_embedding,
        entity_type or 'movie',
        year_min,
        year_max
    )).fetchall()

    if not results or len(results) == 0:
        logger.info(f"No semantic matches above 0.70 for title='{title}'")
        return None

    # Step 3: Return best match
    best = results[0]

    return {
        'canonical_id': best['tmdb_id'] or best['imdb_id'] or best['entity_id'],
        'id_type': 'tmdb_id' if best['tmdb_id'] else 'imdb_id',
        'external_ids': {
            'tmdb_movie': best['tmdb_id'],
            'imdb': best['imdb_id']
        },
        'confidence': float(best['similarity']),
        'tier_used': '3-semantic',
        'metadata': {
            'source': 'vector_search',
            'matched_entity_id': best['entity_id'],
            'matched_title': best['title'],
            'matched_year': best['year'],
            'top_10_matches': [
                {'title': r['title'], 'similarity': r['similarity']}
                for r in results[:10]
            ]
        }
    }
```

### 5.6 Tier 3 Quality Metrics

**Target Metrics:**
- **Precision:** 75%+ (edge cases, lower confidence)
- **Recall:** 70% (handles difficult matches)
- **Latency:** 100-500ms (HNSW index speed)
- **Coverage:** <1% of total queries (last resort tier)

**Monitoring:**
```python
tier3_metrics = {
    'total_queries': 100,          # <1% of total (Tier 2 fallthrough)
    'matches_above_0.85': 30,      # 30% high confidence (review queue)
    'matches_0.70_to_0.84': 50,    # 50% medium confidence (review queue)
    'matches_below_0.70': 20,      # 20% unresolved
    'avg_latency': 180,            # milliseconds
    'p95_latency': 420,            # milliseconds
    'precision_validated': 0.76,   # From manual review sample
    'recall_estimated': 0.72       # From ground truth sample
}
```

---

## 6. Confidence Scoring and Thresholds

### 6.1 Unified Confidence Score

**Confidence Score Definition:**
- Range: 0.0 (no match) to 1.0 (perfect match)
- Interpretation: Probability that two records refer to the same real-world entity

**Tier-Specific Confidence Mapping:**

| Tier | Calculation Method | Typical Range |
|------|-------------------|---------------|
| **Tier 1 (Exact ID)** | Fixed 0.98 (external IDs are highly reliable) | 0.98 |
| **Tier 2 (Fuzzy)** | Fellegi-Sunter match probability | 0.70-0.99 |
| **Tier 3 (Semantic)** | Cosine similarity (normalized) | 0.60-0.95 |

**Confidence Adjustment Factors:**

```python
def adjust_confidence(base_confidence, tier_used, metadata):
    """
    Apply adjustment factors to base confidence score.

    Factors:
        +0.05: Multiple external IDs match (IMDb + TMDB)
        +0.03: Year exact match
        +0.02: Director/cast match
        -0.05: Common title (e.g., "The Office" - ambiguous)
        -0.03: Year mismatch (>2 years apart)
        -0.02: Missing metadata (no year, no type)
    """
    adjusted = base_confidence

    # Boost for multiple ID matches
    if metadata.get('imdb_match') and metadata.get('tmdb_match'):
        adjusted = min(1.0, adjusted + 0.05)

    # Boost for exact year match
    if metadata.get('year_exact'):
        adjusted = min(1.0, adjusted + 0.03)

    # Boost for director/cast match
    if metadata.get('director_match') or metadata.get('cast_overlap'):
        adjusted = min(1.0, adjusted + 0.02)

    # Penalty for common/ambiguous titles
    if is_common_title(metadata.get('title')):
        adjusted = max(0.0, adjusted - 0.05)

    # Penalty for year mismatch
    if metadata.get('year_diff', 0) > 2:
        adjusted = max(0.0, adjusted - 0.03)

    # Penalty for missing metadata
    if not metadata.get('year'):
        adjusted = max(0.0, adjusted - 0.02)

    return round(adjusted, 3)

def is_common_title(title):
    """Check if title is common/ambiguous."""
    common_titles = {
        'the office', 'the wire', 'friends', 'lost', 'the matrix',
        'home alone', 'the thing', 'the fly', 'the holiday'
    }
    return normalize_title(title) in common_titles
```

### 6.2 Decision Thresholds

**Threshold Configuration:**

```python
CONFIDENCE_THRESHOLDS = {
    # Auto-match: High confidence, no human review
    'auto_match': 0.85,

    # Review queue: Medium confidence, human validation recommended
    'review_queue_min': 0.75,
    'review_queue_max': 0.84,

    # Reject: Low confidence, likely not a match
    'reject': 0.75
}
```

**Decision Matrix:**

| Confidence Range | Decision | Action | Rationale |
|------------------|----------|--------|-----------|
| **≥ 0.85** | Auto-match | Cache + return immediately | High precision (90%+), safe to auto-accept |
| **0.75-0.84** | Review queue | Store for manual validation | Medium precision (80-90%), needs human judgment |
| **0.50-0.74** | Possible match | Log for analysis, don't return | Too low precision (<80%), training data only |
| **< 0.50** | Reject | Treat as non-match | Very low precision, likely false positive |

**Threshold Tuning Guidance:**

```python
def recommend_threshold(business_requirement):
    """
    Recommend threshold based on business requirements.

    Args:
        business_requirement: 'precision', 'recall', or 'balanced'
    """
    if business_requirement == 'precision':
        # Minimize false positives (wrong matches)
        # Use case: Canonical ID creation, metadata merging
        return {
            'auto_match': 0.90,
            'review_queue_min': 0.80,
            'reject': 0.80
        }

    elif business_requirement == 'recall':
        # Minimize false negatives (missed matches)
        # Use case: Search/discovery, user-facing recommendations
        return {
            'auto_match': 0.80,
            'review_queue_min': 0.70,
            'reject': 0.70
        }

    else:  # balanced (F1 optimization)
        return CONFIDENCE_THRESHOLDS
```

### 6.3 Conflict Resolution Rules

**Scenario 1: Multiple High-Confidence Matches**

```python
def resolve_multiple_matches(matches):
    """
    Handle case where multiple candidates exceed auto-match threshold.

    Example: "The Office" → US version (2005) vs UK version (2001)
    """
    if len(matches) == 1:
        return matches[0]

    # Rule 1: Prefer exact year match
    exact_year_matches = [m for m in matches if m['metadata'].get('year_exact')]
    if len(exact_year_matches) == 1:
        return exact_year_matches[0]

    # Rule 2: Prefer match with more external IDs
    matches_sorted = sorted(
        matches,
        key=lambda m: len([v for v in m['external_ids'].values() if v]),
        reverse=True
    )
    if matches_sorted[0]['external_ids'] != matches_sorted[1]['external_ids']:
        return matches_sorted[0]

    # Rule 3: Prefer higher tier (1 > 2 > 3)
    tier_priority = {'1-exact': 3, '2-fuzzy': 2, '3-semantic': 1}
    matches_sorted = sorted(
        matches,
        key=lambda m: tier_priority.get(m['tier_used'], 0),
        reverse=True
    )
    if matches_sorted[0]['tier_used'] != matches_sorted[1]['tier_used']:
        return matches_sorted[0]

    # Rule 4: Unable to resolve - send to review queue
    return enqueue_for_review(matches, reason="ambiguous_multiple_matches")
```

**Scenario 2: Conflicting External IDs**

```python
def resolve_id_conflict(match1, match2):
    """
    Handle case where same title has different external IDs.

    Example:
        Match 1: IMDb tt123, TMDB 456
        Match 2: IMDb tt789, TMDB 456
    """
    # Rule 1: If TMDB IDs match but IMDb IDs differ, trust TMDB
    if (match1['external_ids']['tmdb_movie'] == match2['external_ids']['tmdb_movie']
        and match1['external_ids']['imdb'] != match2['external_ids']['imdb']):
        # Flag IMDb ID conflict for manual review
        logger.warning(f"IMDb ID conflict for TMDB {match1['external_ids']['tmdb_movie']}")
        return enqueue_for_review([match1, match2], reason="imdb_id_conflict")

    # Rule 2: If IMDb IDs match but TMDB IDs differ, trust IMDb
    if (match1['external_ids']['imdb'] == match2['external_ids']['imdb']
        and match1['external_ids']['tmdb_movie'] != match2['external_ids']['tmdb_movie']):
        logger.warning(f"TMDB ID conflict for IMDb {match1['external_ids']['imdb']}")
        return enqueue_for_review([match1, match2], reason="tmdb_id_conflict")

    # Rule 3: Different IDs entirely - not the same entity
    return None  # No match
```

---

## 7. SPARQL Query Patterns and Caching

### 7.1 Optimized SPARQL Queries

**Query Optimization Principles:**
1. **Limit results** - Always use LIMIT (Wikidata has millions of items)
2. **Use OPTIONAL sparingly** - Each OPTIONAL adds overhead
3. **Avoid FILTER on large sets** - Use property constraints instead
4. **Batch when possible** - Amortize overhead across 100+ IDs

**Pattern 1: Single ID Lookup (Optimized)**

```sparql
# Optimized for <100ms performance
SELECT ?item ?imdbId ?tmdbMovieId ?tmdbTvId WHERE {
  {
    # Direct property lookup (indexed)
    ?item wdt:P345 "tt0111161" .
  } UNION {
    ?item wdt:P4947 "278" .
  } UNION {
    ?item wdt:P4983 "278" .
  }

  # Fetch all IDs in single query
  OPTIONAL { ?item wdt:P345 ?imdbId . }
  OPTIONAL { ?item wdt:P4947 ?tmdbMovieId . }
  OPTIONAL { ?item wdt:P4983 ?tmdbTvId . }
}
LIMIT 1
```

**Pattern 2: Batch ID Lookup (Cache Refresh)**

```sparql
# Query 100 IDs in single request
SELECT ?imdbId ?item ?tmdbMovieId ?tmdbTvId WHERE {
  VALUES ?imdbId {
    "tt0111161" "tt0068646" "tt0071562" "tt0468569"
    # ... up to 100 IDs
  }
  ?item wdt:P345 ?imdbId .
  OPTIONAL { ?item wdt:P4947 ?tmdbMovieId . }
  OPTIONAL { ?item wdt:P4983 ?tmdbTvId . }
}
```

**Performance:** 500ms-5s for 100 IDs (vs 10-50s for 100 individual queries)

**Pattern 3: Title Search (Fallback - Slow)**

```sparql
# Use only as last resort (risk of timeout)
SELECT DISTINCT ?item ?itemLabel ?imdbId ?tmdbMovieId WHERE {
  VALUES ?type { wd:Q11424 wd:Q5398426 }  # film or TV series
  ?item wdt:P31 ?type .
  ?item rdfs:label ?label .
  FILTER(CONTAINS(LCASE(?label), "shawshank"))
  FILTER(LANG(?label) = "en")

  OPTIONAL { ?item wdt:P345 ?imdbId . }
  OPTIONAL { ?item wdt:P4947 ?tmdbMovieId . }

  SERVICE wikibase:label { bd:serviceParam wikibase:language "en" . }
}
LIMIT 100
```

**Performance:** 1-30s (highly variable)
**Risk:** 60-second timeout[^2]
**Recommendation:** Avoid in production; use Tier 2/3 instead

### 7.2 Cache Management

**Cache Lifecycle:**

```
1. Initial Population (Batch)
   - Fetch 100K most-popular TMDB titles
   - Query Wikidata in batches of 100
   - Store in entity_cache table
   - Duration: ~1.5 hours

2. Real-Time Updates (On-Demand)
   - User queries title without cached mapping
   - Query Wikidata SPARQL (single ID)
   - Cache result with 30-day TTL
   - Duration: <500ms

3. Incremental Refresh (Daily)
   - Find expiring cache entries (expires in <7 days)
   - Batch refresh in groups of 100
   - Update cache_expires_at timestamp
   - Duration: ~30 minutes

4. Invalidation (Manual)
   - Admin detects incorrect mapping
   - DELETE FROM entity_cache WHERE canonical_id = ?
   - Next query triggers fresh SPARQL lookup
```

**Cache Hit Rate Projection:**

| Stage | Cache Size | Cache Hit Rate | Avg Latency |
|-------|------------|----------------|-------------|
| **Day 1 (Empty)** | 0 | 0% | 450ms (all SPARQL) |
| **Day 7 (Organic)** | 5,000 | 20% | 370ms |
| **Day 30 (Batch Pop)** | 100,000 | 90% | 60ms |
| **Day 90 (Mature)** | 150,000 | 95% | 35ms |

**Cache Monitoring:**

```python
def monitor_cache_health():
    """
    Track cache performance metrics.
    """
    metrics = db.execute("""
        SELECT
            COUNT(*) AS total_cached,
            COUNT(*) FILTER (WHERE tier_used = '1-exact') AS tier1_cached,
            COUNT(*) FILTER (WHERE cache_expires_at < NOW() + INTERVAL '7 days') AS expiring_soon,
            AVG(confidence) AS avg_confidence,
            MIN(created_at) AS oldest_entry,
            MAX(last_updated) AS newest_update
        FROM entity_cache
    """).fetchone()

    return {
        'total_cached': metrics['total_cached'],
        'tier1_cached': metrics['tier1_cached'],
        'expiring_soon': metrics['expiring_soon'],
        'avg_confidence': float(metrics['avg_confidence']),
        'cache_age_days': (datetime.now() - metrics['oldest_entry']).days,
        'last_update': metrics['newest_update']
    }
```

---

## 8. Implementation Pseudocode

### 8.1 Master Resolution Function

```python
def resolve_entity(
    tmdb_id: Optional[str] = None,
    imdb_id: Optional[str] = None,
    title: Optional[str] = None,
    year: Optional[int] = None,
    entity_type: Optional[str] = None,
    metadata: Optional[dict] = None
) -> dict:
    """
    Master entity resolution function implementing three-tier pipeline.

    Args:
        tmdb_id: TMDB movie/TV ID
        imdb_id: IMDb ID (tt-prefixed)
        title: Media title
        year: Release/premiere year
        entity_type: 'movie', 'series', or 'documentary'
        metadata: Additional metadata (cast, director, plot)

    Returns:
        dict: {
            'canonical_id': str,       # Wikidata QID or fallback ID
            'id_type': str,            # 'wikidata_qid', 'tmdb_id', 'imdb_id', 'uuid'
            'external_ids': dict,      # All known external IDs
            'confidence': float,       # 0.0-1.0
            'tier_used': str,          # '1-exact', '2-fuzzy', '3-semantic', 'unresolved'
            'metadata': dict,          # Resolution metadata
            'status': str              # 'resolved', 'review_queue', 'unresolved'
        }
    """
    logger.info(f"Resolving entity: tmdb={tmdb_id}, imdb={imdb_id}, title={title}")

    # ========== TIER 1: DETERMINISTIC ID MATCHING ==========
    if tmdb_id or imdb_id:
        result = tier1_exact_match(tmdb_id, imdb_id)
        if result:
            result['status'] = 'resolved'
            logger.info(f"Tier 1 match: {result['canonical_id']} (confidence={result['confidence']})")
            return result
        else:
            logger.info("Tier 1 failed: No external ID match found")

    # ========== TIER 2: PROBABILISTIC FUZZY MATCHING ==========
    if title and year:
        result = tier2_fuzzy_match(title, year, entity_type)

        if result and result['confidence'] >= CONFIDENCE_THRESHOLDS['auto_match']:
            result['status'] = 'resolved'
            logger.info(f"Tier 2 auto-match: {result['canonical_id']} (confidence={result['confidence']})")

            # Cache for future lookups
            cache_entity(result)
            return result

        elif result and result['confidence'] >= CONFIDENCE_THRESHOLDS['review_queue_min']:
            result['status'] = 'review_queue'
            logger.info(f"Tier 2 review queue: {result['canonical_id']} (confidence={result['confidence']})")

            # Store in review queue
            enqueue_for_review(result, reason="medium_confidence_fuzzy_match")
            return result

        else:
            logger.info(f"Tier 2 failed: Best match confidence={result['confidence'] if result else 0.0}")

    # ========== TIER 3: SEMANTIC VECTOR MATCHING ==========
    if title:
        result = tier3_semantic_match(title, year, entity_type)

        if result and result['confidence'] >= 0.70:
            result['status'] = 'review_queue'
            logger.info(f"Tier 3 semantic match: {result['canonical_id']} (confidence={result['confidence']})")

            # All Tier 3 matches go to review queue (lower confidence)
            enqueue_for_review(result, reason="semantic_vector_match")
            return result

        else:
            logger.info(f"Tier 3 failed: Best match confidence={result['confidence'] if result else 0.0}")

    # ========== UNRESOLVED ==========
    logger.warning(f"All tiers failed for title='{title}', tmdb={tmdb_id}, imdb={imdb_id}")

    unresolved_result = create_pending_resolution_task(
        tmdb_id=tmdb_id,
        imdb_id=imdb_id,
        title=title,
        year=year,
        entity_type=entity_type,
        metadata=metadata
    )

    return unresolved_result

def create_pending_resolution_task(
    tmdb_id: Optional[str],
    imdb_id: Optional[str],
    title: Optional[str],
    year: Optional[int],
    entity_type: Optional[str],
    metadata: Optional[dict]
) -> dict:
    """
    Create pending resolution task for unresolved entities.
    """
    # Generate temporary UUID
    import uuid
    temp_id = str(uuid.uuid4())

    # Store in pending_resolution table
    db.execute("""
        INSERT INTO pending_resolution (
            temp_id, tmdb_id, imdb_id, title, year, entity_type,
            metadata, created_at, status
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, NOW(), 'pending')
    """, (temp_id, tmdb_id, imdb_id, title, year, entity_type, json.dumps(metadata or {})))

    return {
        'canonical_id': temp_id,
        'id_type': 'uuid',
        'external_ids': {
            'tmdb_movie': tmdb_id,
            'imdb': imdb_id
        },
        'confidence': 0.0,
        'tier_used': 'unresolved',
        'metadata': {
            'source': 'pending_resolution',
            'title': title,
            'year': year,
            'entity_type': entity_type
        },
        'status': 'unresolved'
    }

def enqueue_for_review(result: dict, reason: str) -> None:
    """
    Add entity to manual review queue.
    """
    db.execute("""
        INSERT INTO review_queue (
            canonical_id, tier_used, confidence, reason,
            metadata, created_at, status
        ) VALUES (%s, %s, %s, %s, %s, NOW(), 'pending_review')
    """, (
        result['canonical_id'],
        result['tier_used'],
        result['confidence'],
        reason,
        json.dumps(result['metadata'])
    ))

    logger.info(f"Added to review queue: {result['canonical_id']} (reason={reason})")

def cache_entity(result: dict) -> None:
    """
    Cache resolved entity for future lookups.
    """
    db.execute("""
        INSERT INTO entity_cache (
            canonical_id, id_type, wikidata_qid,
            tmdb_movie_id, tmdb_tv_id, imdb_id,
            tier_used, confidence, source, metadata,
            cache_expires_at
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, NOW() + INTERVAL '30 days')
        ON CONFLICT (canonical_id) DO UPDATE
        SET last_updated = NOW()
    """, (
        result['canonical_id'],
        result['id_type'],
        result['external_ids'].get('wikidata'),
        result['external_ids'].get('tmdb_movie'),
        result['external_ids'].get('tmdb_tv'),
        result['external_ids'].get('imdb'),
        result['tier_used'],
        result['confidence'],
        result['metadata'].get('source'),
        json.dumps(result['metadata'])
    ))
```

### 8.2 Database Schema

```sql
-- ========== ENTITY CACHE (Tier 1 Results) ==========
CREATE TABLE entity_cache (
    canonical_id VARCHAR(20) PRIMARY KEY,
    id_type VARCHAR(20) NOT NULL,

    wikidata_qid VARCHAR(20) UNIQUE,
    tmdb_movie_id VARCHAR(20),
    tmdb_tv_id VARCHAR(20),
    imdb_id VARCHAR(20),
    tvdb_id VARCHAR(20),
    justwatch_id VARCHAR(50),

    tier_used VARCHAR(20) NOT NULL,
    confidence NUMERIC(4,3) NOT NULL,
    source VARCHAR(50) NOT NULL,

    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    last_updated TIMESTAMP DEFAULT NOW(),
    cache_expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '30 days',

    CONSTRAINT unique_tmdb_movie UNIQUE NULLS NOT DISTINCT (tmdb_movie_id),
    CONSTRAINT unique_tmdb_tv UNIQUE NULLS NOT DISTINCT (tmdb_tv_id),
    CONSTRAINT unique_imdb UNIQUE NULLS NOT DISTINCT (imdb_id)
);

CREATE INDEX idx_entity_cache_tmdb_movie ON entity_cache(tmdb_movie_id) WHERE tmdb_movie_id IS NOT NULL;
CREATE INDEX idx_entity_cache_tmdb_tv ON entity_cache(tmdb_tv_id) WHERE tmdb_tv_id IS NOT NULL;
CREATE INDEX idx_entity_cache_imdb ON entity_cache(imdb_id) WHERE imdb_id IS NOT NULL;
CREATE INDEX idx_entity_cache_expires ON entity_cache(cache_expires_at);

-- ========== ENTITY BLOCKS (Tier 2 Candidates) ==========
CREATE TABLE entity_blocks (
    entity_id VARCHAR(50) PRIMARY KEY,

    title_normalized VARCHAR(500) NOT NULL,
    year INTEGER,
    entity_type VARCHAR(20) NOT NULL,

    block_l1 VARCHAR(50),
    block_l2 VARCHAR(50),
    block_l3 VARCHAR(50),

    imdb_id VARCHAR(20),
    tmdb_id VARCHAR(20),

    cast_list TEXT[],
    director VARCHAR(200),
    plot_summary TEXT,

    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_entity_blocks_l1 ON entity_blocks(block_l1);
CREATE INDEX idx_entity_blocks_l2 ON entity_blocks(block_l2);
CREATE INDEX idx_entity_blocks_l3 ON entity_blocks(block_l3);
CREATE INDEX idx_entity_blocks_year_type ON entity_blocks(year, entity_type);

-- ========== MEDIA EMBEDDINGS (Tier 3 Vector Search) ==========
CREATE TABLE media_embeddings (
    entity_id VARCHAR(50) PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    year INTEGER,
    entity_type VARCHAR(20) NOT NULL,

    imdb_id VARCHAR(20),
    tmdb_id VARCHAR(20),

    embedding FLOAT[384] NOT NULL,

    language VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW()
);

-- HNSW index (DuckDB VSS extension)
CREATE INDEX hnsw_idx ON media_embeddings
USING HNSW (embedding)
WITH (metric = 'cosine', ef_construction = 200, M = 16);

CREATE INDEX idx_media_embeddings_type ON media_embeddings(entity_type);
CREATE INDEX idx_media_embeddings_year ON media_embeddings(year);

-- ========== REVIEW QUEUE (Medium Confidence Matches) ==========
CREATE TABLE review_queue (
    id SERIAL PRIMARY KEY,
    canonical_id VARCHAR(50) NOT NULL,
    tier_used VARCHAR(20) NOT NULL,
    confidence NUMERIC(4,3) NOT NULL,
    reason VARCHAR(100) NOT NULL,

    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    reviewed_at TIMESTAMP,
    reviewer VARCHAR(100),

    status VARCHAR(20) DEFAULT 'pending_review',
    resolution VARCHAR(20),  -- 'approved', 'rejected', 'modified'
    notes TEXT
);

CREATE INDEX idx_review_queue_status ON review_queue(status);
CREATE INDEX idx_review_queue_created ON review_queue(created_at);

-- ========== PENDING RESOLUTION (Unresolved Entities) ==========
CREATE TABLE pending_resolution (
    id SERIAL PRIMARY KEY,
    temp_id VARCHAR(50) UNIQUE NOT NULL,

    tmdb_id VARCHAR(20),
    imdb_id VARCHAR(20),
    title VARCHAR(500),
    year INTEGER,
    entity_type VARCHAR(20),

    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),

    status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'resolved', 'abandoned'
    resolved_canonical_id VARCHAR(50),
    resolved_at TIMESTAMP
);

CREATE INDEX idx_pending_resolution_status ON pending_resolution(status);
CREATE INDEX idx_pending_resolution_created ON pending_resolution(created_at);
```

---

## 9. Quality Metrics and Monitoring

### 9.1 Target Metrics

**Overall System Metrics:**

| Metric | MVP Target | Production Target | Measurement Method |
|--------|------------|-------------------|-------------------|
| **Precision** | 90%+ | 92%+ | Manual validation of random sample (n=500) |
| **Recall** | 85%+ | 88%+ | Ground truth dataset (n=1000 known matches) |
| **F1 Score** | 0.87+ | 0.90+ | Harmonic mean of precision and recall |
| **Coverage** | 95%+ | 97%+ | % of queries resolved (any tier) |
| **Latency (p50)** | <200ms | <100ms | Application performance monitoring |
| **Latency (p95)** | <500ms | <300ms | Application performance monitoring |
| **Latency (p99)** | <2s | <1s | Application performance monitoring |
| **Throughput** | 50 req/sec | 100 req/sec | Load testing |

**Tier-Specific Metrics:**

| Tier | Precision Target | Recall Target | Coverage Target | Latency Target (p95) |
|------|------------------|---------------|-----------------|----------------------|
| **Tier 1 (Exact)** | 98%+ | 92% | 90-93% | <500ms |
| **Tier 2 (Fuzzy)** | 90%+ | 85% | 6-8% | <5s |
| **Tier 3 (Semantic)** | 75%+ | 70% | <1% | <500ms |

### 9.2 Monitoring Dashboard

**Key Performance Indicators (KPIs):**

```python
def calculate_daily_metrics():
    """
    Calculate entity resolution performance metrics.
    """
    today = datetime.now().date()

    metrics = db.execute(f"""
        WITH resolution_stats AS (
            SELECT
                tier_used,
                confidence,
                status,
                EXTRACT(EPOCH FROM (resolved_at - requested_at)) AS latency_sec
            FROM resolution_log
            WHERE DATE(requested_at) = %s
        )
        SELECT
            COUNT(*) AS total_queries,

            -- Coverage by tier
            COUNT(*) FILTER (WHERE tier_used = '1-exact') AS tier1_count,
            COUNT(*) FILTER (WHERE tier_used = '2-fuzzy') AS tier2_count,
            COUNT(*) FILTER (WHERE tier_used = '3-semantic') AS tier3_count,
            COUNT(*) FILTER (WHERE tier_used = 'unresolved') AS unresolved_count,

            -- Status breakdown
            COUNT(*) FILTER (WHERE status = 'resolved') AS resolved_count,
            COUNT(*) FILTER (WHERE status = 'review_queue') AS review_queue_count,
            COUNT(*) FILTER (WHERE status = 'unresolved') AS pending_count,

            -- Confidence distribution
            AVG(confidence) AS avg_confidence,
            PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY confidence) AS median_confidence,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY confidence) AS p95_confidence,

            -- Latency distribution
            AVG(latency_sec) * 1000 AS avg_latency_ms,
            PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY latency_sec) * 1000 AS p50_latency_ms,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_sec) * 1000 AS p95_latency_ms,
            PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_sec) * 1000 AS p99_latency_ms
        FROM resolution_stats
    """, (today,)).fetchone()

    # Calculate derived metrics
    coverage = (metrics['resolved_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0

    tier1_coverage = (metrics['tier1_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0
    tier2_coverage = (metrics['tier2_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0
    tier3_coverage = (metrics['tier3_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0

    return {
        'date': today,
        'total_queries': metrics['total_queries'],

        # Coverage
        'overall_coverage': coverage,
        'tier1_coverage': tier1_coverage,
        'tier2_coverage': tier2_coverage,
        'tier3_coverage': tier3_coverage,
        'unresolved_rate': (metrics['unresolved_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0,

        # Confidence
        'avg_confidence': float(metrics['avg_confidence']),
        'median_confidence': float(metrics['median_confidence']),
        'p95_confidence': float(metrics['p95_confidence']),

        # Latency
        'avg_latency_ms': float(metrics['avg_latency_ms']),
        'p50_latency_ms': float(metrics['p50_latency_ms']),
        'p95_latency_ms': float(metrics['p95_latency_ms']),
        'p99_latency_ms': float(metrics['p99_latency_ms']),

        # Status
        'review_queue_rate': (metrics['review_queue_count'] / metrics['total_queries']) if metrics['total_queries'] > 0 else 0
    }
```

### 9.3 Quality Validation

**Precision/Recall Measurement:**

```python
def measure_precision_recall(sample_size=500):
    """
    Measure precision and recall using ground truth validation set.

    Precision = TP / (TP + FP)  - Of matches returned, how many are correct?
    Recall = TP / (TP + FN)     - Of correct matches, how many did we find?
    """
    # Fetch random sample of resolved entities
    sample = db.execute("""
        SELECT canonical_id, external_ids, confidence, tier_used, metadata
        FROM entity_cache
        ORDER BY RANDOM()
        LIMIT %s
    """, (sample_size,)).fetchall()

    # Manual validation (human review)
    validation_results = []
    for entity in sample:
        # Present to human reviewer
        is_correct = validate_match_with_human(entity)
        validation_results.append({
            'canonical_id': entity['canonical_id'],
            'tier_used': entity['tier_used'],
            'confidence': entity['confidence'],
            'is_correct': is_correct
        })

    # Calculate metrics
    total = len(validation_results)
    correct = sum(1 for r in validation_results if r['is_correct'])

    precision = correct / total if total > 0 else 0

    # For recall, need ground truth dataset of known matches
    # (Measure separately using curated test set)

    # Breakdown by tier
    tier_metrics = {}
    for tier in ['1-exact', '2-fuzzy', '3-semantic']:
        tier_results = [r for r in validation_results if r['tier_used'] == tier]
        tier_correct = sum(1 for r in tier_results if r['is_correct'])
        tier_metrics[tier] = {
            'total': len(tier_results),
            'correct': tier_correct,
            'precision': tier_correct / len(tier_results) if len(tier_results) > 0 else 0
        }

    return {
        'overall_precision': precision,
        'sample_size': total,
        'tier_metrics': tier_metrics
    }
```

**Alerting Rules:**

```python
ALERT_THRESHOLDS = {
    'precision_critical': 0.85,     # Alert if precision drops below 85%
    'coverage_critical': 0.90,      # Alert if coverage drops below 90%
    'p95_latency_warning': 1000,    # Alert if p95 latency exceeds 1s
    'p99_latency_critical': 5000,   # Alert if p99 latency exceeds 5s
    'unresolved_rate_warning': 0.10 # Alert if >10% queries unresolved
}

def check_alerts(metrics):
    """
    Check if metrics violate alert thresholds.
    """
    alerts = []

    if metrics['overall_precision'] < ALERT_THRESHOLDS['precision_critical']:
        alerts.append({
            'severity': 'critical',
            'metric': 'precision',
            'value': metrics['overall_precision'],
            'threshold': ALERT_THRESHOLDS['precision_critical'],
            'message': f"Precision dropped to {metrics['overall_precision']:.2%}"
        })

    if metrics['overall_coverage'] < ALERT_THRESHOLDS['coverage_critical']:
        alerts.append({
            'severity': 'critical',
            'metric': 'coverage',
            'value': metrics['overall_coverage'],
            'threshold': ALERT_THRESHOLDS['coverage_critical'],
            'message': f"Coverage dropped to {metrics['overall_coverage']:.2%}"
        })

    if metrics['p95_latency_ms'] > ALERT_THRESHOLDS['p95_latency_warning']:
        alerts.append({
            'severity': 'warning',
            'metric': 'p95_latency',
            'value': metrics['p95_latency_ms'],
            'threshold': ALERT_THRESHOLDS['p95_latency_warning'],
            'message': f"P95 latency exceeded {metrics['p95_latency_ms']:.0f}ms"
        })

    if metrics['unresolved_rate'] > ALERT_THRESHOLDS['unresolved_rate_warning']:
        alerts.append({
            'severity': 'warning',
            'metric': 'unresolved_rate',
            'value': metrics['unresolved_rate'],
            'threshold': ALERT_THRESHOLDS['unresolved_rate_warning'],
            'message': f"Unresolved rate exceeded {metrics['unresolved_rate']:.2%}"
        })

    return alerts
```

---

## 10. API Specification

### 10.1 REST API Endpoint

**Endpoint:** `POST /api/v1/resolve`

**Request:**

```json
{
  "tmdb_id": "278",
  "imdb_id": "tt0111161",
  "title": "The Shawshank Redemption",
  "year": 1994,
  "entity_type": "movie",
  "metadata": {
    "director": "Frank Darabont",
    "cast": ["Tim Robbins", "Morgan Freeman"],
    "plot": "Two imprisoned men bond over a number of years..."
  }
}
```

**Response (Success - Tier 1):**

```json
{
  "canonical_id": "Q174284",
  "id_type": "wikidata_qid",
  "external_ids": {
    "wikidata": "Q174284",
    "tmdb_movie": "278",
    "imdb": "tt0111161",
    "tvdb": null,
    "justwatch": "tm/movie/278"
  },
  "confidence": 0.98,
  "tier_used": "1-exact",
  "status": "resolved",
  "metadata": {
    "source": "wikidata_sparql",
    "cache_hit": true,
    "resolution_time_ms": 12
  }
}
```

**Response (Review Queue - Tier 2):**

```json
{
  "canonical_id": "278",
  "id_type": "tmdb_id",
  "external_ids": {
    "tmdb_movie": "278",
    "imdb": null
  },
  "confidence": 0.82,
  "tier_used": "2-fuzzy",
  "status": "review_queue",
  "metadata": {
    "source": "splink",
    "matched_title": "shawshank redemption",
    "comparison_scores": {
      "title_similarity": 0.89,
      "year_match": true,
      "type_match": true
    },
    "review_reason": "medium_confidence_fuzzy_match",
    "resolution_time_ms": 2340
  }
}
```

**Response (Unresolved):**

```json
{
  "canonical_id": "550e8400-e29b-41d4-a716-446655440000",
  "id_type": "uuid",
  "external_ids": {},
  "confidence": 0.0,
  "tier_used": "unresolved",
  "status": "unresolved",
  "metadata": {
    "source": "pending_resolution",
    "title": "The Obscure Film",
    "year": 2023,
    "entity_type": "documentary",
    "pending_task_id": 12345,
    "resolution_time_ms": 6780
  }
}
```

### 10.2 Batch Resolution Endpoint

**Endpoint:** `POST /api/v1/resolve/batch`

**Request:**

```json
{
  "entities": [
    {
      "id": "req-1",
      "tmdb_id": "278",
      "title": "The Shawshank Redemption",
      "year": 1994
    },
    {
      "id": "req-2",
      "imdb_id": "tt0068646",
      "title": "The Godfather",
      "year": 1972
    },
    {
      "id": "req-3",
      "title": "The Office",
      "year": 2005,
      "entity_type": "series"
    }
  ]
}
```

**Response:**

```json
{
  "results": [
    {
      "request_id": "req-1",
      "canonical_id": "Q174284",
      "confidence": 0.98,
      "status": "resolved"
    },
    {
      "request_id": "req-2",
      "canonical_id": "Q47703",
      "confidence": 0.98,
      "status": "resolved"
    },
    {
      "request_id": "req-3",
      "canonical_id": "Q13417",
      "confidence": 0.85,
      "status": "resolved"
    }
  ],
  "summary": {
    "total": 3,
    "resolved": 3,
    "review_queue": 0,
    "unresolved": 0,
    "avg_confidence": 0.94,
    "avg_resolution_time_ms": 145
  }
}
```

---

## 11. Deployment and Operations

### 11.1 Infrastructure Requirements

**Minimum Viable Product (100K titles):**

| Component | Specification | Rationale |
|-----------|---------------|-----------|
| **PostgreSQL** | 16+, 8GB RAM, 50GB SSD | Entity cache, blocking keys, review queue |
| **DuckDB** | In-process (embedded) | Splink fuzzy matching, vector search |
| **Application Server** | 4 vCPU, 8GB RAM | Python/Node.js with Sentence Transformers |
| **Redis** | 2GB RAM | Query result caching, rate limiting |
| **SPARQL Client** | HTTP client with timeout | Wikidata API integration |

**Production (1M+ titles):**

| Component | Specification | Rationale |
|-----------|---------------|-----------|
| **PostgreSQL** | 16+, 32GB RAM, 500GB SSD | Larger cache, higher throughput |
| **DuckDB** | Standalone server mode | Distributed query processing |
| **Application Server** | 8 vCPU, 16GB RAM, autoscaling | Handle 100 req/sec sustained |
| **Redis Cluster** | 3 nodes, 8GB RAM each | High availability, session state |
| **Vector Database** | Qdrant/pgvectorscale | Dedicated vector index for Tier 3 |
| **Load Balancer** | Application-level (Nginx/HAProxy) | Distribute traffic, health checks |

### 11.2 Deployment Checklist

**Pre-Deployment:**
- [ ] Populate entity_cache with 100K most-popular titles (batch SPARQL)
- [ ] Populate entity_blocks with normalized titles and blocking keys
- [ ] Generate embeddings for all titles (batch process with Sentence Transformers)
- [ ] Build HNSW index in DuckDB (30-60 seconds for 100K vectors)
- [ ] Configure SPARQL timeout (10 seconds) and retry logic
- [ ] Set up Redis caching layer with 5-minute TTL
- [ ] Configure monitoring dashboards (Grafana + Prometheus)
- [ ] Load test with 1,000 concurrent requests

**Post-Deployment:**
- [ ] Monitor cache hit rate (target 90%+)
- [ ] Monitor resolution latency (p95 <500ms)
- [ ] Monitor unresolved rate (target <5%)
- [ ] Review queue processing (manual validation of medium-confidence matches)
- [ ] Weekly cache refresh job (update expiring entries)
- [ ] Monthly precision/recall validation (n=500 sample)

### 11.3 Operational Runbook

**Common Issues and Resolutions:**

| Issue | Symptoms | Root Cause | Resolution |
|-------|----------|------------|------------|
| **High latency (p95 >1s)** | Slow user queries | SPARQL timeout, cache cold | Increase cache TTL, batch pre-populate cache |
| **Low cache hit rate (<50%)** | Frequent SPARQL calls | Insufficient pre-population | Run batch cache job for top 100K titles |
| **Low precision (<85%)** | User corrections, wrong matches | Threshold too low | Increase auto-match threshold to 0.90 |
| **High unresolved rate (>10%)** | Many pending tasks | Wikidata coverage gap | Lower Tier 2 threshold to 0.70 |
| **SPARQL timeouts (>5%)** | Wikidata API errors | Complex queries, rate limiting | Simplify queries, implement exponential backoff |
| **Review queue backlog** | >1000 pending items | Insufficient manual review | Hire human reviewers, implement active learning |

---

## Sources

[^1]: [WikiData External ID - TMDB Talk](https://www.themoviedb.org/talk/5e639fe63e01ea0015e99824) - TMDB Community, October 2022 - Wikidata coverage statistics

[^2]: [Wikidata SPARQL Entity Resolution Evaluation](/workspaces/research/docs/research/wikidata-sparql-evaluation.md) - A3-WIKIDATA Report, 2025-12-06 - SPARQL performance and timeout limits

[^3]: [TMDB API Evaluation Report](/workspaces/research/docs/research/tmdb-api-evaluation.md) - A1-TMDB Report, 2025-12-06 - TMDB external ID mapping capabilities

[^4]: [The Fellegi-Sunter Model - Splink](https://moj-analytical-services.github.io/splink/topic_guides/theory/fellegi_sunter.html) - UK Ministry of Justice - Probabilistic record linkage theory

[^5]: [Splink Official Documentation](https://moj-analytical-services.github.io/splink/index.html) - UK Ministry of Justice - Production performance benchmarks

[^6]: [Embedding Models and Dimensions](https://devblogs.microsoft.com/azure-sql/embedding-models-and-dimensions-optimizing-the-performance-resource-usage-ratio/) - Microsoft Azure SQL Blog - Embedding model performance comparison

[^7]: [How Big Are Our Embeddings Now](https://vickiboykis.com/2025/09/01/how-big-are-our-embeddings-now-and-why/) - Vicki Boykis, 2025 - Embedding dimension tradeoffs

[^8]: [DuckDB Vector Similarity Search](https://duckdb.org/2024/05/03/vector-similarity-search-vss) - DuckDB Official Blog, May 2024 - HNSW index performance

[^9]: [Pre-trained Embeddings for Entity Resolution (VLDB)](https://www.vldb.org/pvldb/vol16/p2225-skoutas.pdf) - VLDB 2023 - Similarity threshold research

---

**Document Version:** 1.0
**Last Updated:** 2025-12-06
**Author:** C2-ER-SPEC (Entity Resolution Specification Writer)
**Review Status:** Ready for B1-ENTITY and B2-VECTOR review
**Implementation Priority:** CRITICAL (Week 1-2 of hackathon)
