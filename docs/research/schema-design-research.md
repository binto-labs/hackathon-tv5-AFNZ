---
status: keep
phase: complete
type: report
version: 1.0
last-updated: 2025-12-06
title: Schema Design Research - TV5 Media Gateway
agent: B4-SCHEMA
scavs_compliance: true
---

# Schema Design Research - TV5 Media Gateway

## Executive Summary

This report defines a unified metadata schema for TV5 Media Gateway that combines Schema.org (web interoperability), EBUCore (European broadcast standards), and multi-source data fusion patterns. The hybrid PostgreSQL design uses normalized core entities with JSONB flexibility for variable metadata, supporting Wikidata QID as canonical identifier with provenance tracking across TMDB, IMDb, and streaming APIs.

**Key Recommendations:**
- Adopt Schema.org as base vocabulary (TVSeries, Movie, Person types)
- Implement EBUCore alignment for European broadcast partnerships
- Use Wikidata QID (Q-number) as canonical cross-reference identifier
- Store multi-source metadata with quality scoring and provenance tracking
- Deploy hybrid PostgreSQL schema: normalized core + JSONB for variable data

---

## 1. Schema.org Media Types

Schema.org V29.3 (2025-09-04) provides comprehensive media vocabulary optimized for web discovery and interoperability.

### 1.1 Core Media Types Hierarchy

```
Thing
 └─ CreativeWork
     ├─ Movie
     ├─ TVSeries
     │   └─ TVSeason
     │       └─ TVEpisode
     ├─ VideoObject
     └─ CreativeWorkSeries
         └─ MovieSeries
```

### 1.2 Movie Type Properties

**Movie-Specific Properties:**

| Property | Type | Description | TV5 Usage |
|----------|------|-------------|-----------|
| `actor` | Person, PerformingGroup | Cast members | Store with billing order |
| `director` | Person | Film director | Primary creator metadata |
| `duration` | Duration | Runtime in ISO 8601 | `PT2H15M` format |
| `musicBy` | Person, MusicGroup | Composer | Optional enrichment |
| `productionCompany` | Organization | Studio/producer | Map to EBUCore contributor |
| `countryOfOrigin` | Country | Production country | Regional metadata |
| `titleEIDR` | Text, URL | EIDR identifier | External ID cross-reference |
| `trailer` | VideoObject | Preview video | Streaming integration |

**Key Inherited Properties (from CreativeWork):**

| Property | Type | TV5 Usage |
|----------|------|-----------|
| `name` | Text | Primary title (multilingual) |
| `alternateName` | Text | Alternative titles, translations |
| `description` | Text | Plot synopsis (multiple languages) |
| `genre` | Text, URL | Genre taxonomy mapping |
| `datePublished` | Date | Release date |
| `aggregateRating` | AggregateRating | Computed from multiple sources |
| `contentRating` | Text | Age rating (MPAA, PEGI, etc.) |
| `inLanguage` | Language, Text | Original language (BCP 47) |
| `sameAs` | URL | External identifiers (IMDb, Wikidata) |

**Source:** [Schema.org Movie Type](https://schema.org/Movie), [Schema.org Tutorial](https://www.w3resource.com/schema.org/Movie.php)

### 1.3 TVSeries Type Properties

**TVSeries Relationship Structure:**
```json
{
  "@type": "TVSeries",
  "name": "Breaking Bad",
  "containsSeason": [
    {
      "@type": "TVSeason",
      "seasonNumber": 1,
      "episode": [
        {
          "@type": "TVEpisode",
          "episodeNumber": 1,
          "partOfSeason": { "@type": "TVSeason" },
          "partOfSeries": { "@type": "TVSeries" }
        }
      ]
    }
  ]
}
```

**TVSeries-Specific Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `containsSeason` | CreativeWorkSeason | Seasons in series |
| `episode` | Episode | Direct episode references |
| `numberOfSeasons` | Integer | Total season count |
| `numberOfEpisodes` | Integer | Total episode count |
| `startDate` | Date | Series premiere |
| `endDate` | Date | Series finale (if concluded) |

**Nested Structure Benefits:**
- Hierarchical relationships via `hasPart`/`isPartOf`
- Support for both episodic and season-level metadata
- BroadcastEvent and OnDemandEvent for streaming tracking

**Sources:** [Schema.org TVSeries](https://schema.org/TVSeries), [Schema.org CreativeWorkSeries](https://schema.org/CreativeWorkSeries)

### 1.4 VideoObject Integration

VideoObject represents the actual media file/stream and can be embedded in Movie/TVEpisode:

```json
{
  "@type": "TVEpisode",
  "name": "Pilot",
  "associatedMedia": {
    "@type": "VideoObject",
    "contentUrl": "https://stream.tv5.example/video.m3u8",
    "encodingFormat": "application/vnd.apple.mpegurl",
    "duration": "PT45M",
    "thumbnailUrl": "https://cdn.tv5.example/thumb.jpg",
    "uploadDate": "2024-01-15"
  }
}
```

**Note:** Schema.org has no required properties; implementations choose based on use case and consumer requirements (e.g., Google Search features).

**Sources:** [Schema.org VideoObject](https://stackoverflow.com/questions/2075175/is-there-a-standard-schema-for-video-metadata), [Stack Overflow Discussion](https://stackoverflow.com/questions/46429849/schema-org-tvepisode-entries-for-multi-source-playback-page)

---

## 2. EBUCore Alignment

EBUCore (Tech 3293) is the European Broadcasting Union's metadata standard, serving as "Dublin Core for media."

### 2.1 EBUCore Overview

**Purpose:** Minimum metadata set for audiovisual content in broadcast, archive, exchange, and publication contexts.

**Latest Development:** EBUCorePlus v2.0 merges EBUCore and Class Conceptual Data Model (CCDM) into unified ontology for media enterprises, supporting AI-driven services and cross-platform metadata management.

**Adoption:**
- Used by BBC, DR, DW, ORF, and other EBU members
- Integrated with Europeana Data Model (EDM)
- Default metadata for FIMS (Framework of Interoperable Media Services)
- Mapped to PBCore, IPTC Video Metadata Hub, Schema.org

**Licensing:** Creative Commons CC BY-NC-SA 3.0 (freely modifiable)

**Sources:** [EBU Metadata Specifications](https://tech.ebu.ch/metadata/ebucore), [EBUCorePlus v2.0](https://tech.ebu.ch/events/2025/ebucoreplus-v20--the-backbone-of-media-data), [EBU Tech 3293 PDF](https://tech.ebu.ch/docs/tech/tech3293.pdf)

### 2.2 EBUCore to Schema.org Mapping

| EBUCore Element | Schema.org Property | Notes |
|-----------------|---------------------|-------|
| `title` | `name`, `alternateName` | Multiple title variants |
| `description` / `synopsis` | `description` | Plot summary |
| `publicationEvent` | `datePublished`, `BroadcastEvent` | Release/broadcast dates |
| `genre` | `genre` | Controlled vocabulary |
| `contributor` (role: director) | `director` | Role-based mapping |
| `contributor` (role: actor) | `actor` | Cast information |
| `contributor` (role: producer) | `productionCompany` | Production credits |
| `duration` | `duration` | ISO 8601 duration |
| `language` | `inLanguage` | BCP 47 language codes |
| `rating` | `contentRating` | Age/content ratings |
| `identifier` | `identifier`, `sameAs` | External IDs |
| `format` / `technicalAttributes` | `encodingFormat`, VideoObject | Technical metadata |

**Interoperability Note:** EBUCore RDF ontology explicitly maps to Schema.org, W3C Media Annotation ontology, Dublin Core, and FOAF. Properties submitted jointly by EBU and BBC to Schema.org for broadcast publication events.

**Sources:** [EBUCore Schema Documentation](https://www.ebu.ch/metadata/schemas/EBUCore/documentation/urn_ebu_metadata-schema_ebucore.html), [IPTC Video Metadata Hub](https://iptc.org/news/video-metadata-hub-1-3-new-mappings-clarifications-eidr-and-ebucore/)

### 2.3 EBUCore Extensions Beyond Schema.org

EBUCore provides broadcast-specific elements not in Schema.org:

1. **Technical Metadata:**
   - Video codec, bitrate, resolution, aspect ratio
   - Audio tracks, channels, sample rate
   - MXF file embedding (via EBU MXF SDK)
   - BWF-chunk for audio metadata (Tech 3285-s5)

2. **Production Metadata:**
   - Production context (studio, location, equipment)
   - Rights and licensing information
   - Archive/preservation metadata

3. **Editorial Metadata:**
   - Editorial notes, annotations
   - Content classification beyond genre
   - Segmentation and scene markers

**Recommendation for TV5:** Store EBUCore-specific technical metadata in JSONB `technical_metadata` field, maintaining compatibility for European broadcast partnerships while keeping Schema.org as primary web-facing vocabulary.

**Sources:** [EBU Technology & Innovation](https://tech.ebu.ch/metadata), [The Broadcast Bridge Standards Article](https://www.thebroadcastbridge.com/content/entry/20984/standards-part-26-an-introduction-to-metadata)

---

## 3. Unified Schema Draft

### 3.1 Core Entity Types

```typescript
// TypeScript type definitions for TV5 Media Gateway

type CanonicalID = {
  wikidataQID: string;        // Primary canonical ID (e.g., "Q484301")
  wikidataURI: string;        // Full URI: https://www.wikidata.org/entity/Q484301
};

type ExternalIDs = {
  tmdb_id?: number;           // TMDB primary ID
  imdb_id?: string;           // IMDb ID (tt*, nm* prefixes)
  tvdb_id?: number;           // TheTVDB ID
  eidr_id?: string;           // Entertainment ID Registry
  viaf_id?: string;           // Virtual International Authority File
};

type ProvenanceRecord = {
  field: string;              // Field name being tracked
  source: DataSource;         // Which API provided this value
  confidence: number;         // Quality score 0.0-1.0
  timestamp: Date;            // When data was retrieved
  version: string;            // Source API version
};

enum DataSource {
  TMDB = 'tmdb',
  IMDB = 'imdb',
  WIKIDATA = 'wikidata',
  JUSTWATCH = 'justwatch',
  EIDR = 'eidr',
  MANUAL = 'manual'
}

type QualityMetadata = {
  overall_confidence: number; // Aggregate quality score
  completeness: number;       // % of expected fields populated
  freshness_days: number;     // Days since last update
  source_count: number;       // Number of contributing sources
  conflict_count: number;     // Number of conflicting values detected
};

// Base interface for all media entities
interface MediaEntity {
  // Identity
  id: string;                           // Internal UUID
  canonical_id: CanonicalID;            // Wikidata QID (primary)
  external_ids: ExternalIDs;            // Cross-platform IDs

  // Core metadata (Schema.org)
  name: LocalizedText[];                // Primary title
  alternate_names: LocalizedText[];     // Alternative titles
  description: LocalizedText[];         // Synopsis/description

  // Quality & Provenance
  quality: QualityMetadata;
  provenance: ProvenanceRecord[];

  // Versioning
  schema_version: string;               // "1.0"
  last_updated: Date;
  created_at: Date;
}

type LocalizedText = {
  value: string;
  language: string;                     // BCP 47 (en, fr-CA, etc.)
  script?: string;                      // ISO 15924 (Latn, Cyrl, etc.)
  source?: DataSource;
  confidence?: number;
};

interface Title extends MediaEntity {
  // Type discrimination
  media_type: 'movie' | 'tv_series';

  // Descriptive metadata
  genres: string[];                     // Controlled vocabulary
  release_date: Date;
  country_of_origin: string[];          // ISO 3166-1 alpha-2
  original_language: string;            // BCP 47

  // Ratings
  content_ratings: ContentRating[];
  aggregate_rating?: AggregateRating;

  // People & Organizations
  directors: PersonReference[];
  cast: CastMember[];
  production_companies: OrganizationReference[];

  // Media-type specific
  duration?: number;                    // Minutes (for movies)
  number_of_seasons?: number;           // For TV series
  number_of_episodes?: number;          // For TV series

  // Streaming availability
  streaming_availability: StreamingProvider[];

  // Technical metadata (EBUCore)
  technical_metadata?: Record<string, any>; // JSONB

  // Extensions
  custom_metadata?: Record<string, any>;     // JSONB for future fields
}

interface Person extends MediaEntity {
  // Professional names
  birth_name?: string;
  stage_names: string[];

  // Biographical
  birth_date?: Date;
  birth_place?: string;
  death_date?: Date;

  // Roles
  known_for_department?: string;        // Acting, Directing, etc.

  // Profile media
  profile_images: ImageReference[];
}

interface Organization extends MediaEntity {
  organization_type: 'studio' | 'network' | 'distributor' | 'production_company';
  founded_date?: Date;
  headquarters?: string;
  parent_organization?: OrganizationReference;
}

// Supporting types
type ContentRating = {
  rating: string;                       // PG-13, TV-MA, etc.
  system: string;                       // MPAA, TV Parental Guidelines
  country: string;                      // ISO 3166-1
  source?: DataSource;
};

type AggregateRating = {
  rating_value: number;                 // 0-10 scale
  rating_count: number;
  best_rating: number;
  worst_rating: number;
  sources: {
    source: DataSource;
    rating: number;
    count?: number;
  }[];
};

type CastMember = {
  person: PersonReference;
  character_name?: string;
  billing_order: number;                // 0-based cast order
  role_type: 'actor' | 'voice';
};

type PersonReference = {
  id: string;
  canonical_id: CanonicalID;
  name: string;
  external_ids?: ExternalIDs;
};

type OrganizationReference = {
  id: string;
  canonical_id?: CanonicalID;
  name: string;
  external_ids?: ExternalIDs;
};

type StreamingProvider = {
  provider_name: string;
  provider_id: string;
  country: string;                      // ISO 3166-1
  availability_type: 'subscription' | 'rent' | 'buy' | 'free';
  url?: string;
  added_date?: Date;
  source: DataSource;
  last_verified: Date;
};

type ImageReference = {
  url: string;
  width: number;
  height: number;
  aspect_ratio: number;
  type: 'poster' | 'backdrop' | 'still' | 'profile' | 'logo';
  language?: string;
  source: DataSource;
};
```

### 3.2 Schema.org JSON-LD Example

```json
{
  "@context": "https://schema.org",
  "@type": "TVSeries",
  "@id": "https://tv5.example/series/breaking-bad",
  "sameAs": [
    "https://www.wikidata.org/entity/Q1079",
    "https://www.imdb.com/title/tt0903747/",
    "https://www.themoviedb.org/tv/1396"
  ],
  "identifier": [
    {
      "@type": "PropertyValue",
      "propertyID": "WIKIDATA_QID",
      "value": "Q1079"
    },
    {
      "@type": "PropertyValue",
      "propertyID": "TMDB_ID",
      "value": "1396"
    },
    {
      "@type": "PropertyValue",
      "propertyID": "IMDB_ID",
      "value": "tt0903747"
    }
  ],
  "name": "Breaking Bad",
  "alternateName": [
    "Breaking Bad: La Química del Mal",
    "Во все тяжкие"
  ],
  "description": "A high school chemistry teacher turned methamphetamine producer partners with a former student.",
  "genre": ["Crime", "Drama", "Thriller"],
  "numberOfSeasons": 5,
  "numberOfEpisodes": 62,
  "startDate": "2008-01-20",
  "endDate": "2013-09-29",
  "inLanguage": "en-US",
  "countryOfOrigin": {
    "@type": "Country",
    "name": "United States"
  },
  "contentRating": "TV-MA",
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": 9.5,
    "bestRating": 10,
    "ratingCount": 2100000
  },
  "actor": [
    {
      "@type": "Person",
      "@id": "https://tv5.example/person/bryan-cranston",
      "sameAs": "https://www.wikidata.org/entity/Q23833",
      "name": "Bryan Cranston"
    }
  ],
  "director": [
    {
      "@type": "Person",
      "sameAs": "https://www.wikidata.org/entity/Q5871",
      "name": "Vince Gilligan"
    }
  ],
  "productionCompany": {
    "@type": "Organization",
    "name": "AMC Studios"
  }
}
```

---

## 4. Cross-Reference ID Pattern

### 4.1 Wikidata QID as Canonical Identifier

**Rationale:**
- **Persistent:** Wikidata QIDs never change (Q484301 is always "The Simpsons")
- **Universal:** Language-agnostic, globally recognized
- **Linked Data:** Connects to VIAF, Library of Congress, IMDb, TMDB
- **Authority:** Community-curated with versioned edits
- **Integration:** Apple Siri, Amazon Alexa, Google Knowledge Graph use Wikidata

**QID Structure:**
- Format: `Q` + positive integer (e.g., Q1079, Q23833)
- URI: `https://www.wikidata.org/entity/{QID}`
- Used as globally unique identifier across systems

**Sources:** [Wikidata Identifiers](https://www.wikidata.org/wiki/Wikidata:Identifiers), [Wikidata Wikipedia Article](https://en.wikipedia.org/wiki/Wikidata), [Wikidata for Authority Control](https://www.wikidata.org/wiki/Wikidata:Wikidata_for_authority_control)

### 4.2 Multi-Source ID Resolution Strategy

```
┌─────────────────────────────────────────────────┐
│  Phase 1: ID Collection from Sources           │
├─────────────────────────────────────────────────┤
│  TMDB API → tmdb_id, imdb_id, wikidata_qid     │
│  IMDb → imdb_id                                 │
│  Wikidata SPARQL → QID, imdb_id, tmdb_id       │
│  JustWatch → streaming provider IDs             │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│  Phase 2: Canonical ID Resolution              │
├─────────────────────────────────────────────────┤
│  1. Check if Wikidata QID exists in any source │
│  2. If yes → Use as canonical_id                │
│  3. If no → Query Wikidata by IMDb/TMDB ID     │
│  4. If found → Store QID as canonical           │
│  5. If not found → Flag for manual review      │
└─────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────┐
│  Phase 3: Store All IDs with Provenance        │
├─────────────────────────────────────────────────┤
│  canonical_id: { wikidataQID: "Q1079" }        │
│  external_ids: {                                │
│    tmdb_id: 1396,                               │
│    imdb_id: "tt0903747",                        │
│    tvdb_id: 81189                               │
│  }                                              │
│  provenance: [                                  │
│    { field: "tmdb_id", source: "tmdb", ... }   │
│    { field: "wikidata_qid", source: "tmdb" }   │
│  ]                                              │
└─────────────────────────────────────────────────┘
```

### 4.3 Wikidata Integration Code Pattern

```typescript
// Entity resolution service
class EntityResolver {
  async resolveCanonicalID(
    tmdbId?: number,
    imdbId?: string,
    wikidataQID?: string
  ): Promise<CanonicalID | null> {

    // Priority 1: Use provided Wikidata QID if available
    if (wikidataQID) {
      return {
        wikidataQID,
        wikidataURI: `https://www.wikidata.org/entity/${wikidataQID}`
      };
    }

    // Priority 2: Query Wikidata by IMDb ID (most reliable)
    if (imdbId) {
      const qid = await this.queryWikidataByIMDb(imdbId);
      if (qid) return this.buildCanonicalID(qid);
    }

    // Priority 3: Query Wikidata by TMDB ID
    if (tmdbId) {
      const qid = await this.queryWikidataByTMDB(tmdbId);
      if (qid) return this.buildCanonicalID(qid);
    }

    // No canonical ID found
    return null;
  }

  private async queryWikidataByIMDb(imdbId: string): Promise<string | null> {
    const sparql = `
      SELECT ?item WHERE {
        ?item wdt:P345 "${imdbId}".
      }
      LIMIT 1
    `;

    const result = await this.wikidataSPARQL(sparql);
    return result?.item?.replace('http://www.wikidata.org/entity/', '');
  }

  private async queryWikidataByTMDB(tmdbId: number): Promise<string | null> {
    // P4947 = TMDB movie ID, P4983 = TMDB TV series ID
    const sparql = `
      SELECT ?item WHERE {
        { ?item wdt:P4947 "${tmdbId}". }
        UNION
        { ?item wdt:P4983 "${tmdbId}". }
      }
      LIMIT 1
    `;

    const result = await this.wikidataSPARQL(sparql);
    return result?.item?.replace('http://www.wikidata.org/entity/', '');
  }
}
```

**Coverage Note:** Per TMDB evaluation, only 7-10% of TMDB entries currently have Wikidata QIDs mapped. TV5 gateway should implement fallback patterns and flag unmapped entities for manual review.

**Sources:** [TMDB Wikidata Integration](https://www.themoviedb.org/talk/614c6a12e1faed002d1d0099), [Wikidata SPARQL Guide](https://stackoverflow.com/questions/74256381/how-to-get-the-wikidata-id-q-using-just-a-label)

---

## 5. Provenance Tracking

### 5.1 Provenance Data Model

**Definition:** Data provenance tracks the history and origin of each data point, documenting which source provided which value, when, and with what confidence level.

**W3C PROV Standard:** Industry-standard framework for documenting provenance on the web, ensuring interoperability between systems.

**Sources:** [What is Data Provenance - IBM](https://www.ibm.com/think/topics/data-provenance), [Data Provenance - Acceldata](https://www.acceldata.io/blog/data-provenance), [W3C PROV](https://en.wikipedia.org/wiki/Data_lineage)

### 5.2 Field-Level Provenance Tracking

```typescript
interface ProvenanceRecord {
  field: string;                  // "title.en", "genres", "release_date"
  source: DataSource;             // TMDB, IMDB, WIKIDATA
  value: any;                     // The actual value from this source
  confidence: number;             // 0.0 - 1.0
  timestamp: Date;                // When retrieved
  version: string;                // API version
  retrieval_method: string;       // "api", "scrape", "manual"
}

// Example: Title field with multiple sources
const titleProvenance = [
  {
    field: "name.en",
    source: DataSource.TMDB,
    value: "Breaking Bad",
    confidence: 0.95,
    timestamp: new Date("2025-12-06T10:00:00Z"),
    version: "3.0",
    retrieval_method: "api"
  },
  {
    field: "name.en",
    source: DataSource.IMDB,
    value: "Breaking Bad",
    confidence: 0.98,
    timestamp: new Date("2025-12-06T10:05:00Z"),
    version: "2024",
    retrieval_method: "api"
  },
  {
    field: "name.en",
    source: DataSource.WIKIDATA,
    value: "Breaking Bad",
    confidence: 1.0,  // Community-curated, high trust
    timestamp: new Date("2025-12-06T10:10:00Z"),
    version: "wikibase-1.39",
    retrieval_method: "sparql"
  }
];
```

### 5.3 Confidence Scoring Algorithm

**Multi-Source Fusion Approaches:**

1. **Voting Methods:** Majority voting when sources agree
2. **Weighted Fusion:** Source reliability weights × confidence scores
3. **Statistical Methods:** Bayesian belief networks for quality estimation
4. **Discord Analysis:** Measure discordance to identify conflicting data

**Source Reliability Weights (Initial Configuration):**

```typescript
const SOURCE_WEIGHTS = {
  WIKIDATA: 1.0,      // Community-curated, versioned
  IMDB: 0.95,         // Industry standard, high quality
  TMDB: 0.90,         // Large dataset, good coverage
  JUSTWATCH: 0.85,    // Streaming data specialist
  EIDR: 0.95,         // Industry registry
  MANUAL: 0.70        // Human entry, subject to error
};

function calculateConfidence(
  provenanceRecords: ProvenanceRecord[]
): number {

  // Case 1: Single source
  if (provenanceRecords.length === 1) {
    return provenanceRecords[0].confidence;
  }

  // Case 2: Multiple sources agree (exact match)
  const uniqueValues = new Set(provenanceRecords.map(p => p.value));
  if (uniqueValues.size === 1) {
    // Agreement bonus: average of weighted confidences
    const weightedSum = provenanceRecords.reduce(
      (sum, p) => sum + (p.confidence * SOURCE_WEIGHTS[p.source]),
      0
    );
    const weightSum = provenanceRecords.reduce(
      (sum, p) => sum + SOURCE_WEIGHTS[p.source],
      0
    );
    return Math.min(0.99, weightedSum / weightSum); // Cap at 0.99
  }

  // Case 3: Conflict - use highest weighted confidence
  const bestSource = provenanceRecords.reduce((best, current) => {
    const currentScore = current.confidence * SOURCE_WEIGHTS[current.source];
    const bestScore = best.confidence * SOURCE_WEIGHTS[best.source];
    return currentScore > bestScore ? current : best;
  });

  // Conflict penalty: reduce confidence by 20%
  return bestSource.confidence * SOURCE_WEIGHTS[bestSource.source] * 0.8;
}
```

**Quality Metadata Aggregation:**

```typescript
function computeQualityMetadata(
  entity: MediaEntity,
  provenanceRecords: ProvenanceRecord[]
): QualityMetadata {

  const expectedFields = getExpectedFields(entity.media_type);
  const populatedFields = countPopulatedFields(entity);

  // Group by field to detect conflicts
  const byField = groupBy(provenanceRecords, 'field');
  const conflictCount = Object.values(byField).filter(
    records => new Set(records.map(r => r.value)).size > 1
  ).length;

  // Compute overall confidence (average of field confidences)
  const fieldConfidences = Object.values(byField).map(
    records => calculateConfidence(records)
  );
  const overallConfidence = average(fieldConfidences);

  // Freshness: days since oldest record
  const oldestTimestamp = Math.min(
    ...provenanceRecords.map(p => p.timestamp.getTime())
  );
  const freshnessDays = daysSince(oldestTimestamp);

  return {
    overall_confidence: overallConfidence,
    completeness: populatedFields / expectedFields,
    freshness_days: freshnessDays,
    source_count: new Set(provenanceRecords.map(p => p.source)).size,
    conflict_count: conflictCount
  };
}
```

**Sources:** [Multi-Source Data Fusion](https://www.sciencedirect.com/topics/computer-science/multi-sensor-data-fusion), [Quality-Based Score Fusion](https://dl.acm.org/doi/10.1145/3377644.3377647), [Evaluating Disparate Data Sources](https://link.springer.com/chapter/10.1007/978-3-032-05281-0_10)

### 5.4 Conflict Resolution Strategy

**When sources disagree:**

1. **Automatic Resolution (Low Stakes):**
   - Genre tags: Union of all sources
   - Alternative titles: Store all variants
   - Release dates: Use earliest verified date

2. **Weighted Resolution (Medium Stakes):**
   - Descriptions: Use longest from highest-weighted source
   - Cast order: Prefer IMDB (industry standard for billing)
   - Ratings: Compute weighted average

3. **Manual Review (High Stakes):**
   - Primary title discrepancies
   - Director/creator conflicts
   - Major factual discrepancies (e.g., runtime ±30 minutes)

```typescript
enum ConflictResolutionStrategy {
  UNION,              // Combine all values (genres, tags)
  HIGHEST_CONFIDENCE, // Use most confident source
  WEIGHTED_AVERAGE,   // Average with source weights
  MANUAL_REVIEW,      // Flag for human review
  LONGEST_VALUE,      // Use most detailed (descriptions)
  EARLIEST_DATE       // Use oldest verified date
}

const FIELD_STRATEGIES = {
  'genres': ConflictResolutionStrategy.UNION,
  'description': ConflictResolutionStrategy.LONGEST_VALUE,
  'name': ConflictResolutionStrategy.MANUAL_REVIEW,
  'release_date': ConflictResolutionStrategy.EARLIEST_DATE,
  'aggregate_rating': ConflictResolutionStrategy.WEIGHTED_AVERAGE,
  'cast_order': ConflictResolutionStrategy.HIGHEST_CONFIDENCE
};
```

**Sources:** [Data Quality Monitoring](https://www.secoda.co/blog/importance-of-data-provenance), [Knowledge Fusion Trust Scoring](https://blog.diffbot.com/knowledge-graph-glossary/data-provenance/)

---

## 6. Database Implementation

### 6.1 PostgreSQL Hybrid Schema

**Design Philosophy:** Normalize queryable core attributes, use JSONB for variable/optional metadata.

**Rationale:**
- B-tree indexes on normalized columns are 10-100x faster than GIN indexes
- JSONB provides schema flexibility for evolving metadata
- Foreign keys enforce referential integrity for relationships
- JSONB supports European EBUCore technical metadata without rigid schema

**Sources:** [PostgreSQL JSONB vs Normalized](https://stackoverflow.com/questions/36894395/performance-of-json-vs-metadata-table), [AWS PostgreSQL as JSON Database](https://aws.amazon.com/blogs/database/postgresql-as-a-json-database-advanced-patterns-and-best-practices/), [PostgreSQL JSONB Architecture Weekly](https://www.architecture-weekly.com/p/postgresql-jsonb-powerful-storage)

### 6.2 Database Schema SQL

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Trigram fuzzy search

-- Enum types
CREATE TYPE media_type_enum AS ENUM ('movie', 'tv_series', 'tv_season', 'tv_episode');
CREATE TYPE data_source_enum AS ENUM ('tmdb', 'imdb', 'wikidata', 'justwatch', 'eidr', 'manual');
CREATE TYPE availability_type_enum AS ENUM ('subscription', 'rent', 'buy', 'free');

-- ============================================================================
-- CORE ENTITIES
-- ============================================================================

-- Titles table (movies and TV series)
CREATE TABLE titles (
  -- Identity
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  canonical_wikidata_qid VARCHAR(20),  -- Q1079, Q484301, etc.
  canonical_wikidata_uri TEXT,         -- Full Wikidata URI

  -- Type
  media_type media_type_enum NOT NULL,

  -- External IDs (indexed for lookups)
  tmdb_id INTEGER,
  imdb_id VARCHAR(20),
  tvdb_id INTEGER,
  eidr_id VARCHAR(50),

  -- Core metadata (normalized for performance)
  primary_title TEXT NOT NULL,
  primary_language VARCHAR(10) NOT NULL,  -- BCP 47
  original_language VARCHAR(10),
  release_date DATE,
  duration_minutes INTEGER,              -- For movies
  number_of_seasons INTEGER,             -- For TV series
  number_of_episodes INTEGER,            -- For TV series

  -- Ratings (normalized for query performance)
  aggregate_rating_value NUMERIC(3,1),  -- 0.0-10.0
  aggregate_rating_count INTEGER,
  content_rating VARCHAR(20),            -- PG-13, TV-MA, etc.

  -- Quality metadata
  overall_confidence NUMERIC(3,2),       -- 0.00-1.00
  completeness NUMERIC(3,2),             -- 0.00-1.00
  freshness_days INTEGER,
  source_count INTEGER,
  conflict_count INTEGER,

  -- Flexible metadata (JSONB)
  localized_titles JSONB,                -- [{ value, language, source }]
  localized_descriptions JSONB,          -- [{ value, language, source }]
  genres JSONB,                          -- ["Drama", "Crime"]
  countries_of_origin JSONB,             -- ["US", "GB"]
  content_ratings JSONB,                 -- [{ rating, system, country }]
  rating_sources JSONB,                  -- [{ source, rating, count }]
  technical_metadata JSONB,              -- EBUCore technical data
  custom_metadata JSONB,                 -- Future extensions

  -- Provenance tracking
  provenance JSONB NOT NULL,             -- Array of ProvenanceRecord

  -- Versioning
  schema_version VARCHAR(10) NOT NULL DEFAULT '1.0',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  last_updated TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  -- Indexes inline
  CONSTRAINT unique_wikidata_qid UNIQUE (canonical_wikidata_qid),
  CONSTRAINT unique_tmdb_id UNIQUE (tmdb_id, media_type),
  CONSTRAINT unique_imdb_id UNIQUE (imdb_id)
);

-- People table (actors, directors, crew)
CREATE TABLE people (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  canonical_wikidata_qid VARCHAR(20),
  canonical_wikidata_uri TEXT,

  -- External IDs
  tmdb_id INTEGER UNIQUE,
  imdb_id VARCHAR(20) UNIQUE,

  -- Core metadata
  primary_name TEXT NOT NULL,
  birth_name TEXT,
  birth_date DATE,
  birth_place TEXT,
  death_date DATE,
  known_for_department VARCHAR(50),

  -- Flexible metadata
  localized_names JSONB,                 -- Stage names, translations
  biography JSONB,                       -- Localized biographies
  profile_images JSONB,
  custom_metadata JSONB,

  -- Provenance
  provenance JSONB NOT NULL,
  overall_confidence NUMERIC(3,2),

  schema_version VARCHAR(10) NOT NULL DEFAULT '1.0',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  last_updated TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  CONSTRAINT unique_person_wikidata_qid UNIQUE (canonical_wikidata_qid)
);

-- Organizations table (studios, networks, production companies)
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  canonical_wikidata_qid VARCHAR(20),
  canonical_wikidata_uri TEXT,

  -- External IDs
  tmdb_id INTEGER UNIQUE,

  -- Core metadata
  organization_type VARCHAR(50) NOT NULL,
  primary_name TEXT NOT NULL,
  founded_date DATE,
  headquarters TEXT,
  parent_organization_id UUID REFERENCES organizations(id),

  -- Flexible metadata
  localized_names JSONB,
  custom_metadata JSONB,

  -- Provenance
  provenance JSONB NOT NULL,
  overall_confidence NUMERIC(3,2),

  schema_version VARCHAR(10) NOT NULL DEFAULT '1.0',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  last_updated TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  CONSTRAINT unique_org_wikidata_qid UNIQUE (canonical_wikidata_qid)
);

-- ============================================================================
-- RELATIONSHIP TABLES
-- ============================================================================

-- Title-Person relationships (cast, crew)
CREATE TABLE title_credits (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  title_id UUID NOT NULL REFERENCES titles(id) ON DELETE CASCADE,
  person_id UUID NOT NULL REFERENCES people(id) ON DELETE CASCADE,

  role_type VARCHAR(50) NOT NULL,        -- 'actor', 'director', 'writer', etc.
  character_name TEXT,                   -- For actors
  billing_order INTEGER,                 -- Cast order (0-based)

  -- Provenance for this relationship
  source data_source_enum NOT NULL,
  confidence NUMERIC(3,2),

  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  CONSTRAINT unique_title_person_role UNIQUE (title_id, person_id, role_type, character_name)
);

CREATE INDEX idx_title_credits_title ON title_credits(title_id);
CREATE INDEX idx_title_credits_person ON title_credits(person_id);
CREATE INDEX idx_title_credits_role ON title_credits(role_type);

-- Title-Organization relationships (production companies)
CREATE TABLE title_organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  title_id UUID NOT NULL REFERENCES titles(id) ON DELETE CASCADE,
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  role_type VARCHAR(50) NOT NULL,        -- 'production', 'distribution', etc.

  source data_source_enum NOT NULL,
  confidence NUMERIC(3,2),

  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  CONSTRAINT unique_title_org UNIQUE (title_id, organization_id, role_type)
);

CREATE INDEX idx_title_orgs_title ON title_organizations(title_id);
CREATE INDEX idx_title_orgs_org ON title_organizations(organization_id);

-- Streaming availability
CREATE TABLE streaming_availability (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  title_id UUID NOT NULL REFERENCES titles(id) ON DELETE CASCADE,

  provider_name TEXT NOT NULL,
  provider_id TEXT NOT NULL,
  country VARCHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
  availability_type availability_type_enum NOT NULL,
  url TEXT,
  added_date DATE,

  source data_source_enum NOT NULL,
  last_verified TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  CONSTRAINT unique_title_provider_country UNIQUE (title_id, provider_id, country)
);

CREATE INDEX idx_streaming_title ON streaming_availability(title_id);
CREATE INDEX idx_streaming_provider ON streaming_availability(provider_name);
CREATE INDEX idx_streaming_country ON streaming_availability(country);

-- ============================================================================
-- PERFORMANCE INDEXES
-- ============================================================================

-- B-tree indexes for normalized columns (fastest lookups)
CREATE INDEX idx_titles_media_type ON titles(media_type);
CREATE INDEX idx_titles_release_date ON titles(release_date);
CREATE INDEX idx_titles_rating ON titles(aggregate_rating_value) WHERE aggregate_rating_value IS NOT NULL;
CREATE INDEX idx_titles_primary_title ON titles(primary_title);
CREATE INDEX idx_titles_wikidata ON titles(canonical_wikidata_qid) WHERE canonical_wikidata_qid IS NOT NULL;

-- Trigram indexes for fuzzy text search
CREATE INDEX idx_titles_primary_title_trgm ON titles USING gin(primary_title gin_trgm_ops);
CREATE INDEX idx_people_primary_name_trgm ON people USING gin(primary_name gin_trgm_ops);

-- GIN indexes for JSONB (use jsonb_path_ops for better performance)
CREATE INDEX idx_titles_genres ON titles USING gin(genres jsonb_path_ops);
CREATE INDEX idx_titles_localized_titles ON titles USING gin(localized_titles jsonb_path_ops);
CREATE INDEX idx_titles_provenance ON titles USING gin(provenance jsonb_path_ops);

-- Composite indexes for common queries
CREATE INDEX idx_titles_type_release ON titles(media_type, release_date DESC);
CREATE INDEX idx_titles_type_rating ON titles(media_type, aggregate_rating_value DESC NULLS LAST);

-- ============================================================================
-- FUNCTIONS & TRIGGERS
-- ============================================================================

-- Auto-update last_updated timestamp
CREATE OR REPLACE FUNCTION update_last_updated()
RETURNS TRIGGER AS $$
BEGIN
  NEW.last_updated = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER titles_last_updated
  BEFORE UPDATE ON titles
  FOR EACH ROW
  EXECUTE FUNCTION update_last_updated();

CREATE TRIGGER people_last_updated
  BEFORE UPDATE ON people
  FOR EACH ROW
  EXECUTE FUNCTION update_last_updated();

CREATE TRIGGER organizations_last_updated
  BEFORE UPDATE ON organizations
  FOR EACH ROW
  EXECUTE FUNCTION update_last_updated();

-- ============================================================================
-- VIEWS FOR COMMON QUERIES
-- ============================================================================

-- Materialized view for high-quality titles
CREATE MATERIALIZED VIEW high_quality_titles AS
SELECT
  id,
  canonical_wikidata_qid,
  media_type,
  primary_title,
  release_date,
  aggregate_rating_value,
  overall_confidence,
  completeness
FROM titles
WHERE
  overall_confidence >= 0.80
  AND completeness >= 0.70
  AND conflict_count = 0
ORDER BY aggregate_rating_value DESC NULLS LAST;

CREATE INDEX idx_hq_titles_rating ON high_quality_titles(aggregate_rating_value DESC NULLS LAST);
CREATE INDEX idx_hq_titles_type ON high_quality_titles(media_type);

-- Refresh strategy: daily or on-demand
-- REFRESH MATERIALIZED VIEW CONCURRENTLY high_quality_titles;
```

### 6.3 Index Strategy Rationale

**B-tree Indexes (Normalized Columns):**
- Fastest for exact lookups and range queries
- Used for: primary keys, foreign keys, dates, ratings
- Performance: O(log n) lookup time

**GIN Indexes (JSONB Columns):**
- Use `jsonb_path_ops` operator class for containment queries (@>)
- 16% slower than unindexed for `jsonb_path_ops` vs 79% for default `jsonb_ops`
- Only support Bitmap Index Scans (not Index Scan)
- Trade-off: Larger write overhead for flexible querying

**Trigram Indexes (Full-Text Search):**
- Enable fuzzy matching for titles and names
- Example: "Breaking Bad" matches "breaking bd", "braking bad"
- Uses `pg_trgm` extension

**When to Use Which:**
```sql
-- Fast: B-tree on normalized column
SELECT * FROM titles WHERE release_date >= '2020-01-01';

-- Medium: GIN on JSONB with jsonb_path_ops
SELECT * FROM titles WHERE genres @> '["Drama"]'::jsonb;

-- Fuzzy: Trigram for text search
SELECT * FROM titles WHERE primary_title % 'braking bad';  -- Similarity search

-- Avoid: Querying deep JSONB paths without expression index
-- Slow: SELECT * FROM titles WHERE localized_titles->0->>'value' = 'Breaking Bad';
-- Better: Create expression index if this query is common
CREATE INDEX idx_titles_first_localized
  ON titles ((localized_titles->0->>'value'));
```

**Sources:** [Understanding Postgres GIN Indexes](https://pganalyze.com/blog/gin-index), [PostgreSQL JSONB Operator Classes](https://medium.com/@josef.machytka/postgresql-jsonb-operator-classes-of-gin-indexes-and-their-usage-0bf399073a4c), [Indexing JSONB in Postgres](https://www.crunchydata.com/blog/indexing-jsonb-in-postgres)

### 6.4 Query Examples

```sql
-- Find title by Wikidata QID (canonical lookup)
SELECT * FROM titles
WHERE canonical_wikidata_qid = 'Q1079';

-- Find title by TMDB ID with type
SELECT * FROM titles
WHERE tmdb_id = 1396 AND media_type = 'tv_series';

-- Search titles by fuzzy name match
SELECT primary_title, similarity(primary_title, 'braking bad') AS sim
FROM titles
WHERE primary_title % 'braking bad'
ORDER BY sim DESC
LIMIT 10;

-- High-rated dramas released after 2020
SELECT
  primary_title,
  release_date,
  aggregate_rating_value,
  genres
FROM titles
WHERE
  media_type = 'movie'
  AND release_date >= '2020-01-01'
  AND genres @> '["Drama"]'::jsonb
  AND aggregate_rating_value >= 8.0
ORDER BY aggregate_rating_value DESC
LIMIT 20;

-- Get title with full cast (join)
SELECT
  t.primary_title,
  p.primary_name,
  tc.role_type,
  tc.character_name,
  tc.billing_order
FROM titles t
JOIN title_credits tc ON t.id = tc.title_id
JOIN people p ON tc.person_id = p.id
WHERE t.canonical_wikidata_qid = 'Q1079'
  AND tc.role_type = 'actor'
ORDER BY tc.billing_order;

-- Streaming availability by country
SELECT
  t.primary_title,
  sa.provider_name,
  sa.availability_type,
  sa.url
FROM titles t
JOIN streaming_availability sa ON t.id = sa.title_id
WHERE
  t.imdb_id = 'tt0903747'
  AND sa.country = 'US'
  AND sa.last_verified > NOW() - INTERVAL '30 days'
ORDER BY sa.availability_type;

-- Data quality report
SELECT
  media_type,
  COUNT(*) as total,
  AVG(overall_confidence) as avg_confidence,
  AVG(completeness) as avg_completeness,
  AVG(source_count) as avg_sources,
  SUM(CASE WHEN conflict_count > 0 THEN 1 ELSE 0 END) as conflicted_titles
FROM titles
GROUP BY media_type;

-- Provenance audit: which sources provided what
SELECT
  primary_title,
  jsonb_array_elements(provenance) as provenance_record
FROM titles
WHERE canonical_wikidata_qid = 'Q1079';
```

---

## 7. Schema Evolution Strategy

### 7.1 Versioning Approach

**Semantic Versioning for Schema:**
- **MAJOR** (1.x.x → 2.x.x): Breaking changes requiring migration
- **MINOR** (x.1.x → x.2.x): Backward-compatible additions
- **PATCH** (x.x.1 → x.x.2): Bug fixes, clarifications

**Schema Version Field:** Every entity has `schema_version` field (e.g., "1.0", "1.1", "2.0")

**Sources:** [Schema Versioning Azure Cosmos DB](https://devblogs.microsoft.com/cosmosdb/azure-cosmos-db-design-patterns-part-9-schema-versioning/), [Schema Versioning Strategies](https://www.jusdb.com/blog/schema-versioning-and-migration-strategies-for-scalable-databases)

### 7.2 Backward Compatibility Patterns

**1. Additive Changes Only (Minor Version):**
```sql
-- Safe: Add new optional column
ALTER TABLE titles ADD COLUMN new_field TEXT;

-- Safe: Add new JSONB property (application level)
-- No migration needed, just update application to populate it
```

**2. Dual-Write Pattern (Major Version):**
```typescript
// Phase 1: Write to both old and new schema
async function saveTitleV1andV2(title: Title) {
  await saveToV1Schema(title);
  await saveToV2Schema(title);
}

// Phase 2: Migrate background data
// Phase 3: Switch reads to V2
// Phase 4: Deprecate V1 writes
```

**3. Default Values for New Fields:**
```sql
-- Make new columns nullable with defaults
ALTER TABLE titles ADD COLUMN popularity_score NUMERIC(5,2) DEFAULT 0.0;

-- Application can gradually backfill values
UPDATE titles SET popularity_score = calculate_popularity(id)
WHERE popularity_score = 0.0 AND last_updated < NOW() - INTERVAL '1 week';
```

**4. Schema Version Coexistence:**
```typescript
// Application handles multiple schema versions
function parseTitle(row: any): Title {
  switch (row.schema_version) {
    case '1.0':
      return parseTitleV1(row);
    case '1.1':
      return parseTitleV1_1(row);  // Minor changes
    case '2.0':
      return parseTitleV2(row);    // Major rewrite
    default:
      throw new Error(`Unsupported schema version: ${row.schema_version}`);
  }
}
```

**Sources:** [Database Design Backward Compatibility](https://www.pingcap.com/article/database-design-patterns-for-ensuring-backward-compatibility/), [Schema Evolution Patterns](https://dev3lop.com/schema-evolution-patterns-with-backward-forward-compatibility/)

### 7.3 Migration Strategy

**Phased Rollout:**

1. **Phase 0 - Planning:**
   - Document breaking changes
   - Create migration scripts (tested on staging)
   - Communicate timeline to stakeholders

2. **Phase 1 - Schema Safe (N-1 version compatibility):**
   ```sql
   -- Add new nullable columns
   ALTER TABLE titles ADD COLUMN new_structure JSONB;

   -- Application vN-1 ignores new column but doesn't fail
   ```

3. **Phase 2 - Dual Write (N version):**
   ```typescript
   // Application vN writes to both old and new fields
   await db.query(`
     UPDATE titles
     SET
       old_field = $1,
       new_structure = $2
     WHERE id = $3
   `, [oldValue, newValue, id]);
   ```

4. **Phase 3 - Background Migration:**
   ```sql
   -- Gradual backfill with rate limiting
   UPDATE titles
   SET new_structure = migrate_to_new_format(old_field)
   WHERE new_structure IS NULL
   LIMIT 1000;  -- Run in batches
   ```

5. **Phase 4 - Switch Reads:**
   ```typescript
   // Application vN+1 reads from new_structure
   const title = await fetchTitleV2(id);
   ```

6. **Phase 5 - Deprecate Old:**
   ```sql
   -- After all data migrated and verified
   ALTER TABLE titles DROP COLUMN old_field;
   ```

**Tools:**
- **Flyway** or **Liquibase** for versioned migrations
- **Online Schema Change:** `CREATE INDEX CONCURRENTLY` for zero-downtime
- **Rollback Strategy:** Keep old columns until migration verified (90+ days)

**Sources:** [Schema Evolution Best Practices](https://dataengineeracademy.com/module/best-practices-for-managing-schema-evolution-in-data-pipelines/), [Database Migrations Real World](https://blog.jetbrains.com/idea/2025/02/database-migrations-in-the-real-world/)

### 7.4 JSONB Evolution Benefits

**JSONB provides natural schema evolution:**

```typescript
// Version 1.0: Basic metadata
{
  "technical_metadata": {
    "resolution": "1920x1080",
    "codec": "H.264"
  }
}

// Version 1.1: Add HDR support (backward compatible)
{
  "technical_metadata": {
    "resolution": "1920x1080",
    "codec": "H.264",
    "hdr": "HDR10"  // New field, old parsers ignore
  }
}

// Version 2.0: Restructure (breaking, requires migration)
{
  "technical_metadata": {
    "video": {
      "resolution": { "width": 1920, "height": 1080 },
      "codec": "H.264",
      "hdr": { "format": "HDR10", "max_nits": 1000 }
    },
    "audio": {
      "codec": "AAC",
      "channels": "5.1"
    }
  }
}
```

**Application gracefully handles unknown fields:**
```typescript
// Robust parsing ignores unknown fields
const tech = title.technical_metadata;
const resolution = tech?.resolution || tech?.video?.resolution || 'unknown';
```

---

## 8. Recommendations

| Aspect | Recommendation | Rationale |
|--------|----------------|-----------|
| **Base Vocabulary** | Adopt Schema.org (Movie, TVSeries, Person) | Web standard, SEO benefits, 140+ properties, used by Google/Bing/Apple |
| **European Broadcast** | Implement EBUCore alignment layer | Enables BBC, DR, ORF partnerships; store technical metadata in JSONB |
| **Canonical ID** | Use Wikidata QID as primary identifier | Persistent, universal, authority-controlled; connects to VIAF, Library of Congress |
| **External IDs** | Store TMDB, IMDb, TVDB, EIDR in separate fields | Fast lookups via B-tree indexes; enables multi-API entity resolution |
| **Database Design** | Hybrid: Normalized core + JSONB flexible | 10-100x faster queries on normalized; JSONB for variable metadata |
| **Indexing** | B-tree (normalized), GIN jsonb_path_ops (JSONB), Trigram (search) | Optimized for common query patterns; balance read vs write performance |
| **Provenance** | Field-level tracking with confidence scores | Trust scoring, conflict detection, audit trail, data quality monitoring |
| **Quality Scoring** | Weighted fusion with source reliability weights | Wikidata 1.0, IMDb 0.95, TMDB 0.90; conflict penalty 0.8x |
| **Multi-Language** | LocalizedText array with BCP 47 codes | Support 140+ countries; store translations from all sources |
| **Schema Versioning** | Semantic versioning with phased migrations | Backward compatibility; dual-write during transitions; JSONB flexibility |
| **Missing Wikidata** | Implement fallback + manual review queue | Only 7-10% have QIDs; query Wikidata by IMDb/TMDB; flag unresolved |
| **Conflict Resolution** | Strategy per field type (union/weighted/manual) | Genres: union; descriptions: longest; titles: manual review |
| **Streaming Data** | Separate table with country-level granularity | 600+ providers, 140+ countries; verify freshness (<30 days) |
| **Migration Tools** | Flyway/Liquibase + background batch processing | Version-controlled migrations; zero-downtime online schema changes |
| **Performance** | Materialized views for high-quality subset | Pre-filter confidence ≥0.80, completeness ≥0.70; refresh daily |

---

## 9. Implementation Roadmap

### Phase 1: Core Schema (Week 1-2)
- [ ] Create PostgreSQL database with core tables (titles, people, organizations)
- [ ] Implement B-tree indexes on normalized columns
- [ ] Set up GIN indexes on JSONB fields
- [ ] Write basic CRUD operations with provenance tracking

### Phase 2: TMDB Integration (Week 2-3)
- [ ] Build TMDB API client with rate limiting (50 req/sec)
- [ ] Implement entity resolver (TMDB ID → Wikidata QID)
- [ ] Create provenance tracking for TMDB source
- [ ] Backfill 1000 popular titles for testing

### Phase 3: Wikidata Integration (Week 3-4)
- [ ] Build Wikidata SPARQL client
- [ ] Implement QID resolution by IMDb/TMDB IDs
- [ ] Create background job to enrich missing QIDs
- [ ] Build manual review queue for unmatched entities

### Phase 4: Quality Scoring (Week 4-5)
- [ ] Implement confidence calculation algorithm
- [ ] Build conflict detection system
- [ ] Create data quality dashboard
- [ ] Set up automated quality reports

### Phase 5: Multi-Source Fusion (Week 5-6)
- [ ] Add IMDb integration (if feasible)
- [ ] Add JustWatch for streaming data
- [ ] Implement weighted fusion for conflicting data
- [ ] Create conflict resolution workflows

### Phase 6: Schema.org Export (Week 6)
- [ ] Build JSON-LD generator for Schema.org
- [ ] Create public API endpoints
- [ ] Implement EBUCore RDF export (optional)
- [ ] Performance optimization and caching

---

## 10. Sources

### Schema.org
- [Schema.org Movie Type](https://schema.org/Movie)
- [Schema.org TVSeries Type](https://schema.org/TVSeries)
- [Schema.org CreativeWorkSeries](https://schema.org/CreativeWorkSeries)
- [Schema.org Tutorial - Movie](https://www.w3resource.com/schema.org/Movie.php)
- [Stack Overflow: Video Metadata Standards](https://stackoverflow.com/questions/2075175/is-there-a-standard-schema-for-video-metadata)
- [Stack Overflow: TVEpisode Multi-Source Playback](https://stackoverflow.com/questions/46429849/schema-org-tvepisode-entries-for-multi-source-playback-page)

### EBUCore
- [EBU Metadata Specifications](https://tech.ebu.ch/metadata/ebucore)
- [EBUCorePlus v2.0 Release](https://tech.ebu.ch/events/2025/ebucoreplus-v20--the-backbone-of-media-data)
- [EBU Tech 3293 PDF Specification](https://tech.ebu.ch/docs/tech/tech3293.pdf)
- [EBUCore Schema Documentation](https://www.ebu.ch/metadata/schemas/EBUCore/documentation/urn_ebu_metadata-schema_ebucore.html)
- [The Broadcast Bridge: Metadata Standards](https://www.thebroadcastbridge.com/content/entry/20984/standards-part-26-an-introduction-to-metadata)
- [IPTC Video Metadata Hub 1.3](https://iptc.org/news/video-metadata-hub-1-3-new-mappings-clarifications-eidr-and-ebucore/)
- [EBU Technology & Innovation - Metadata](https://tech.ebu.ch/metadata)

### Wikidata Integration
- [Wikidata Identifiers](https://www.wikidata.org/wiki/Wikidata:Identifiers)
- [Wikidata Wikipedia Article](https://en.wikipedia.org/wiki/Wikidata)
- [Wikidata for Authority Control](https://www.wikidata.org/wiki/Wikidata:Wikidata_for_authority_control)
- [Stack Overflow: Get QID from Label](https://stackoverflow.com/questions/74256381/how-to-get-the-wikidata-id-q-using-just-a-label)
- [Wikipedia: Finding Wikidata ID](https://en.wikipedia.org/wiki/Wikipedia:Finding_a_Wikidata_ID)

### TMDB API
- [TMDB Getting Started](https://developer.themoviedb.org/reference/intro/getting-started)
- [TMDB API Documentation](https://developer.themoviedb.org/docs/getting-started)
- [Stack Overflow: TMDB Genre Mapping](https://stackoverflow.com/questions/72154929/get-genre-name-corresponding-to-genre-id-in-tmdb-api)

### PostgreSQL & Database Design
- [Stack Overflow: JSONB vs Metadata Table Performance](https://stackoverflow.com/questions/36894395/performance-of-json-vs-metadata-table)
- [Database Administrators: JSONB vs Normalized](https://dba.stackexchange.com/questions/221955/postgres-jsonb-column-or-standard-normalized-table)
- [AWS: PostgreSQL as JSON Database](https://aws.amazon.com/blogs/database/postgresql-as-a-json-database-advanced-patterns-and-best-practices/)
- [Architecture Weekly: PostgreSQL JSONB](https://www.architecture-weekly.com/p/postgresql-jsonb-powerful-storage)
- [Understanding Postgres GIN Indexes](https://pganalyze.com/blog/gin-index)
- [How to Index JSONB in PostgreSQL](https://www.tigerdata.com/learn/how-to-index-json-columns-in-postgresql)
- [PostgreSQL JSONB Operator Classes](https://medium.com/@josef.machytka/postgresql-jsonb-operator-classes-of-gin-indexes-and-their-usage-0bf399073a4c)
- [Crunchy Data: Indexing JSONB](https://www.crunchydata.com/blog/indexing-jsonb-in-postgres)
- [PostgreSQL Official Docs: GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
- [PostgreSQL Official Docs: JSON Types](https://www.postgresql.org/docs/current/datatype-json.html)

### Data Provenance & Quality
- [IBM: What is Data Provenance](https://www.ibm.com/think/topics/data-provenance)
- [Acceldata: Tracking Data Provenance](https://www.acceldata.io/blog/data-provenance)
- [Wikipedia: Data Lineage](https://en.wikipedia.org/wiki/Data_lineage)
- [Secoda: Importance of Data Provenance](https://www.secoda.co/blog/importance-of-data-provenance)
- [Diffbot: Data Provenance Glossary](https://blog.diffbot.com/knowledge-graph-glossary/data-provenance/)
- [Meta Broadcast: Competitive Advantage with Provenance](https://www.metabroadcast.com/2024/11/13/gain-a-competitive-advantage-with-data-provenance/)
- [ScienceDirect: Multi-Sensor Data Fusion](https://www.sciencedirect.com/topics/computer-science/multi-sensor-data-fusion)
- [ACM: Quality-Based Score Level Fusion](https://dl.acm.org/doi/10.1145/3377644.3377647)
- [Springer: Evaluating Disparate Data Sources](https://link.springer.com/chapter/10.1007/978-3-032-05281-0_10)
- [PMC: Multimodal Data Fusion Techniques](https://pmc.ncbi.nlm.nih.gov/articles/PMC10007548/)

### Schema Versioning & Migration
- [PingCAP: Database Design for Backward Compatibility](https://www.pingcap.com/article/database-design-patterns-for-ensuring-backward-compatibility/)
- [Data Engineer Academy: Schema Evolution Best Practices](https://dataengineeracademy.com/module/best-practices-for-managing-schema-evolution-in-data-pipelines/)
- [Dev3lop: Schema Evolution Patterns](https://dev3lop.com/schema-evolution-patterns-with-backward-forward-compatibility/)
- [JusDB: Schema Versioning Strategies](https://www.jusdb.com/blog/schema-versioning-and-migration-strategies-for-scalable-databases/)
- [Azure Cosmos DB: Schema Versioning Pattern](https://devblogs.microsoft.com/cosmosdb/azure-cosmos-db-design-patterns-part-9-schema-versioning/)
- [JetBrains: Database Migrations in Real World](https://blog.jetbrains.com/idea/2025/02/database-migrations-in-the-real-world/)
- [Confluent: Schema Evolution and Compatibility](https://docs.confluent.io/cloud/current/sr/fundamentals/schema-evolution.html)
- [Aliz: Data Backwards Compatibility](https://www.aliz.ai/en/blog/data-backwards-compatibility-evolving-your-database-with-no-downtime)

---

**Report completed by B4-SCHEMA research agent**
**Last updated: 2025-12-06**
**SCAVS Compliance: ✅ Sourced, Current, Actionable, Verified, Synthesized**
