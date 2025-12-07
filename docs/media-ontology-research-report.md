# Media Ontology Research Report
## Comprehensive Analysis of Industry Metadata Standards for Unified Media Platform

**Date:** 2025-12-06
**Researcher:** Research Agent
**Purpose:** Evaluate existing ontologies/standards for movies, TV, anime, and future media expansion

---

## Executive Summary

This report evaluates 10+ major media ontology standards and metadata schemas used in the entertainment industry. The research identifies three viable paths forward:

1. **Hybrid Approach (RECOMMENDED)** - Adopt Schema.org as the public-facing foundation with EBUCore extensions for professional features
2. **Professional-First** - Start with EBUCore and map to other standards
3. **Custom Ontology** - Build proprietary schema with mappings to industry standards

**Key Finding:** No single standard covers all requirements. A hybrid approach combining Schema.org (for web discovery) + EBUCore (for professional metadata) + custom extensions (for unique features) offers the best balance of interoperability, completeness, and flexibility.

---

## 1. Comprehensive Ontology Comparison

### 1.1 Comparison Matrix

| Standard | Media Coverage | Adoption | Complexity | Extensibility | License | Best For |
|----------|---------------|----------|------------|---------------|---------|----------|
| **Schema.org** | Movies, TV, Music, Games, Podcasts | Very High (Google, Bing) | Low | Medium | Free (CC) | Web discovery, SEO |
| **EBUCore** | Video, Audio, Broadcast | High (Europe) | High | High | Free (Open) | Professional broadcast, archives |
| **EIDR** | Movies, TV, Shows | High (USA studios) | Medium | Low | Paid registry | Content identification |
| **ISAN** | Movies, TV | Medium (Europe) | Medium | Low | Paid registry | Rights management |
| **MovieLabs OMC** | Film production workflow | Medium (Studios) | Very High | High | Free (Open) | Production pipelines |
| **W3C Media Ont** | Generic media | Medium | Low | High | Free (W3C) | Semantic web integration |
| **TV-Anytime** | Broadcast TV, VOD | Medium (DVB) | High | Medium | Free (ETSI) | Personal video recorders |
| **Wikidata** | All media types | High (Wikipedia) | Low | Very High | Free (CC0) | Knowledge graphs |
| **Dublin Core** | Generic resources | Very High | Very Low | Low | Free | Basic cataloging |
| **IMDb Schema** | Movies, TV, Games | Very High | Medium | None | Proprietary | De facto reference |

---

### 1.2 Detailed Analysis by Standard

#### **Schema.org Movie/TVSeries Vocabulary**
- **Website:** https://schema.org/Movie, https://schema.org/TVSeries
- **Coverage:** Movies, TV Series, Episodes, Music, Games (partial), Podcasts
- **Key Properties:**
  - `name`, `description`, `genre`, `datePublished`
  - `director`, `actor`, `author` (writer), `producer`, `musicBy`
  - `duration`, `aggregateRating`, `trailer`
  - `countryOfOrigin`, `productionCompany`
  - `containsSeason`, `numberOfEpisodes` (TV)

- **Strengths:**
  - Recognized by Google, Bing, Yandex for rich snippets
  - Simple JSON-LD implementation
  - Active community and updates
  - Good for SEO and web discovery

- **Weaknesses:**
  - Limited genre taxonomy (no standardized genre list)
  - No mood/theme descriptors
  - Minimal content rating support
  - US-centric terminology ("season" vs "series")

- **Adoption:** Google Search, Bing, Yandex, Apple, IMDb uses schema.org markup
- **License:** Creative Commons Attribution-ShareAlike License
- **Recommendation:** **Essential foundation for web presence**

---

#### **EBUCore (EBU Tech 3293)**
- **Website:** https://tech.ebu.ch/metadata/ebucore
- **Coverage:** Video, Audio, Broadcast media - comprehensive professional metadata
- **Current Version:** 1.10 (2024)
- **Key Features:**
  - Extension of Dublin Core with 100+ media-specific properties
  - ITU-R BS.2076 Audio Data Model (ADM) integration
  - Scene-level metadata (props, costumes, timed text, actions, emotions)
  - Character and person associations
  - Technical metadata (codecs, resolution, bitrate, color space)
  - Rights and licensing metadata

- **Strengths:**
  - Most comprehensive professional media metadata
  - RDF/OWL representation available
  - Mapped to Schema.org, Dublin Core, FOAF
  - Published as Linked Open Vocabulary (LOV)
  - Tools available (MINT mapper, MXF SDK, MediaInfo)

- **Weaknesses:**
  - High complexity (steep learning curve)
  - Overkill for basic consumer applications
  - Broadcast-focused (may have irrelevant fields)

- **Adoption:** European broadcasters (BBC, ZDF), archives, production companies
- **License:** Free and open (EBU public license)
- **Recommendation:** **Best choice for professional-grade metadata and archive quality**

---

#### **EIDR (Entertainment Identifier Registry)**
- **Website:** https://www.eidr.org/
- **Coverage:** Movies, TV shows, episodes, edits, encodings
- **Technical Foundation:** DOI (Digital Object Identifier) - ISO 26324
- **Key Features:**
  - Unique global identifiers for audiovisual works
  - 2M+ records (400K movies, 1M TV episodes, 40K series)
  - Metadata includes title types, alternate IDs, relationships
  - Interoperable with ISAN, ISRC, Ad-ID

- **Strengths:**
  - Industry standard in USA (encouraged by government)
  - Persistent identifiers for content tracking
  - Rich alternate ID mapping
  - Dual registration with ISAN possible

- **Weaknesses:**
  - Registration fees required
  - Limited to identification (not rich metadata)
  - Less adoption outside USA

- **Adoption:** Major Hollywood studios, streaming platforms (Netflix, Amazon, Disney+)
- **License:** Paid registry (membership fees)
- **Recommendation:** **Use as external identifier field, not primary schema**

---

#### **ISAN (International Standard Audiovisual Number)**
- **Website:** https://www.isan.org/
- **Coverage:** Movies, TV programs (rights management focus)
- **Technical Foundation:** ISO 15706-1:2001, ISO 15706-2:2007
- **Key Features:**
  - ISO standard for audiovisual work identification
  - Created 2002, endorsed by European Union
  - Focus on digital rights management (DRM)
  - Interoperable with EIDR

- **Strengths:**
  - ISO certification and EU endorsement
  - Strong in European market
  - Rights-holder focused

- **Weaknesses:**
  - Registration fees
  - Less known in USA
  - Limited metadata beyond identification

- **Adoption:** European broadcasters, rights organizations, some global studios
- **License:** Paid registry
- **Recommendation:** **Optional external identifier for European distribution**

---

#### **MovieLabs Ontology for Media Creation (OMC)**
- **Website:** https://movielabs.com/ontology-for-media-creation-2/
- **Coverage:** Film/TV production workflows, creative works
- **Current Version:** v2.6 (2024), v2.8+ planned for 2025
- **Key Features:**
  - Production-focused (on-set data, video pipelines, audio)
  - Connected to Ontology for Media Distribution (OMD)
  - Creative Works Ontology for titles, people, locations, awards
  - RDF and JSON schemas available
  - Beta audio ontology included

- **Strengths:**
  - Industry collaboration (SMPTE, EBU, MovieLabs)
  - Semantic web technology (OWL, RDF)
  - Active development (major releases in 2025)
  - Production-to-distribution coverage

- **Weaknesses:**
  - Very complex (professional filmmaking focus)
  - Overkill for consumer media discovery
  - Still evolving (beta components)

- **Adoption:** Major studios, production companies, post-production facilities
- **License:** Free and open
- **Recommendation:** **Consider for future production workflow integration only**

---

#### **W3C Ontology for Media Resources 1.0**
- **Website:** https://www.w3.org/TR/mediaont-10/
- **Coverage:** Generic media resources (video, audio, images)
- **Technical Foundation:** OWL2 DL ontology, RDF/OWL implementation
- **Namespace:** http://www.w3.org/ns/ma-ont#
- **Key Features:**
  - Core vocabulary to bridge different metadata formats
  - Mappings to Dublin Core, MPEG-7, EBUCore, TV-Anytime
  - Semantic Web and Linked Data compatible
  - Generic properties: title, description, creator, contributor, date, language, location

- **Strengths:**
  - W3C Recommendation (official standard)
  - Excellent for semantic web integration
  - Format-agnostic mappings
  - Linked Open Data compatible

- **Weaknesses:**
  - Very generic (lacks media-specific richness)
  - No genre/mood taxonomies
  - Designed for interoperability, not completeness

- **Adoption:** Archives, digital libraries, semantic web projects
- **License:** W3C Document License (free)
- **Recommendation:** **Use as RDF/semantic web foundation if needed**

---

#### **TV-Anytime (ETSI TS 102 822)**
- **Website:** https://dvb.org/metadata/, ETSI TS 102 822 series
- **Coverage:** Broadcast TV, personal video recorders (PVR), video-on-demand
- **Key Features:**
  - Phase 1: Uni-directional broadcast metadata (attractors, user preferences)
  - Phase 2: Home network distribution, network digital recorders
  - XML Schema representation
  - Integration with MPEG-7 description schemes
  - User preference and history metadata

- **Strengths:**
  - Designed for consumer discovery ("attractors")
  - User preference modeling
  - MPEG-7 integration
  - Publication event mechanisms adapted to Schema.org

- **Weaknesses:**
  - Focused on broadcast/PVR use cases
  - Complex for simple streaming platforms
  - Less relevant for on-demand-only services

- **Adoption:** European DVB broadcasters, PVR manufacturers
- **License:** ETSI open standards (free)
- **Recommendation:** **Consider for user preference/recommendation features**

---

#### **Wikidata Film Ontology**
- **Website:** https://www.wikidata.org/wiki/Wikidata:WikiProject_Movies
- **Coverage:** All media types (movies, TV, music, games, podcasts, books)
- **Key Properties (Selection):**
  - `director` (P57), `cast member` (P161), `genre` (P136)
  - `publication date` (P577), `country of origin` (P495)
  - `producer` (P162), `screenwriter` (P58), `based on` (P144)
  - `filming location` (P915), `narrative location` (P840)
  - `duration` (P2047), `IMDb ID` (P345), `EIDR` (P2768)

- **Strengths:**
  - Multilingual (300+ languages)
  - Massive existing dataset (millions of items)
  - Community-maintained and growing
  - CC0 public domain license
  - SPARQL query support
  - Links to IMDb, EIDR, ISAN, and other IDs

- **Weaknesses:**
  - Quality varies (community-edited)
  - Not designed for real-time applications
  - Ontology inconsistencies in some areas
  - May lack specific properties for niche needs

- **Adoption:** Wikipedia, academic research, knowledge graphs
- **License:** CC0 Public Domain
- **Recommendation:** **Excellent data source for seeding initial dataset**

---

#### **Dublin Core Metadata**
- **Website:** https://www.dublincore.org/
- **Coverage:** Generic resources (books, media, documents, websites)
- **15 Core Elements:** Title, Creator, Subject, Description, Publisher, Contributor, Date, Type, Format, Identifier, Source, Language, Relation, Coverage, Rights
- **Key Features:**
  - Simplest possible metadata (15 fields)
  - Resource discovery independent of medium
  - Extended by EBUCore for media

- **Strengths:**
  - Universal adoption in libraries and archives
  - Extremely simple to implement
  - Format-agnostic
  - Compatible with XMP, Premiere Pro, archival systems

- **Weaknesses:**
  - Too generic for rich media metadata
  - No media-specific fields (genre, cast, crew, etc.)
  - Must be extended for useful media cataloging

- **Adoption:** Nearly universal in digital libraries and archives
- **License:** Free and open
- **Recommendation:** **Use as minimal baseline, extend with media-specific standards**

---

#### **IMDb Data Model (De Facto Standard)**
- **Website:** https://www.imdb.com/interfaces/, https://developer.imdb.com/
- **Coverage:** Movies, TV, games, cast, crew
- **Schema Structure:** 8-15 relational tables
  - **Core tables:** `movie`, `person`, `genre`, `language`, `country`, `company`, `role`, `certificate`
  - **Junction tables:** `castinfo`, `movietogenre`, `movie_companies`, etc.

- **Key Features:**
  - Title metadata: ID, title, year, runtime, genres, rating
  - Credits: Cast, crew, roles (normalized to person table)
  - Ratings: 1-10 star ratings from 1 billion+ users
  - Rich metadata: Plots, keywords, posters, trailers
  - 8M+ titles, 11M+ people in licensed dataset

- **Strengths:**
  - De facto industry reference (most complete dataset)
  - Rich genre taxonomy and keyword system
  - User ratings and reviews at scale
  - Proven schema for relational databases

- **Weaknesses:**
  - Proprietary (must license for commercial use)
  - Not normalized in raw TSV export
  - Limited technical metadata (no codec, resolution, etc.)
  - USA-centric in some areas

- **Adoption:** Universal reference, used by all streaming platforms
- **License:** Licensed data (AWS Data Exchange), free non-commercial TSV exports
- **Recommendation:** **Use as reference for schema design and data quality benchmark**

---

## 2. Ontology Requirements Analysis

### 2.1 Content Metadata Requirements

| Requirement | Schema.org | EBUCore | EIDR | W3C Media | TV-Anytime | Wikidata | IMDb | Custom Needed? |
|-------------|-----------|---------|------|-----------|------------|----------|------|----------------|
| **Core Identification** |
| Title (original, localized) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | No |
| Release date/year | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | No |
| Runtime/duration | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | No |
| **Genre & Classification** |
| Genre (standardized) | ⚠️ | ✅ | ❌ | ⚠️ | ✅ | ⚠️ | ✅ | **Yes** |
| Sub-genres | ❌ | ⚠️ | ❌ | ❌ | ⚠️ | ⚠️ | ⚠️ | **Yes** |
| Moods/themes | ❌ | ⚠️ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ | **Yes** |
| Content ratings | ⚠️ | ✅ | ❌ | ❌ | ✅ | ⚠️ | ✅ | No |
| **Credits** |
| Director | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | No |
| Cast | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | No |
| Writers | ✅ | ✅ | ❌ | ✅ | ⚠️ | ✅ | ✅ | No |
| Producers | ✅ | ✅ | ❌ | ✅ | ⚠️ | ✅ | ✅ | No |
| Crew roles | ⚠️ | ✅ | ❌ | ⚠️ | ⚠️ | ⚠️ | ✅ | No |
| **Relationships** |
| Sequel/prequel | ❌ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ | ⚠️ | **Yes** |
| Spinoff | ❌ | ⚠️ | ⚠️ | ❌ | ❌ | ⚠️ | ⚠️ | **Yes** |
| Remake/adaptation | ⚠️ | ⚠️ | ❌ | ❌ | ❌ | ✅ | ⚠️ | **Yes** |
| Based on (source) | ⚠️ | ✅ | ❌ | ⚠️ | ⚠️ | ✅ | ⚠️ | No |
| **Technical Specs** |
| Resolution | ❌ | ✅ | ❌ | ⚠️ | ⚠️ | ❌ | ❌ | No |
| Audio format | ❌ | ✅ | ❌ | ⚠️ | ⚠️ | ❌ | ❌ | No |
| Aspect ratio | ❌ | ✅ | ❌ | ⚠️ | ❌ | ⚠️ | ❌ | No |
| **User Engagement** |
| Ratings | ✅ | ⚠️ | ❌ | ❌ | ⚠️ | ❌ | ✅ | No |
| Reviews | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | No |
| User preferences | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | **Yes** |

**Legend:**
✅ = Full support
⚠️ = Partial support
❌ = Not supported

---

### 2.2 Custom Properties Required

Based on the analysis, the following properties need custom definitions:

#### **Genre & Mood Taxonomy**
```yaml
custom_properties:
  genre_primary:
    type: controlled_vocabulary
    values: [Action, Drama, Comedy, Horror, Sci-Fi, Fantasy, Thriller, Romance, Documentary, Animation, Crime, Mystery, Adventure, Family, War, Western, Musical, Biography, Sport, History]

  genre_secondary:
    type: controlled_vocabulary_array
    description: "Sub-genres or hybrid genres"

  mood_tags:
    type: controlled_vocabulary_array
    values: [Dark, Uplifting, Suspenseful, Heartwarming, Intense, Lighthearted, Thought-Provoking, Gritty, Whimsical, Nostalgic, etc.]

  themes:
    type: controlled_vocabulary_array
    values: [Coming of Age, Redemption, Revenge, Love, Justice, Survival, Identity, Family, Betrayal, etc.]
```

#### **Content Attributes (Parents & Content Rating)**
```yaml
  content_violence:
    type: enum
    values: [None, Mild, Moderate, Severe]

  content_language:
    type: enum
    values: [None, Mild, Moderate, Severe]

  content_sexual:
    type: enum
    values: [None, Mild, Moderate, Severe]

  content_frightening:
    type: enum
    values: [None, Mild, Moderate, Severe]

  content_substance_use:
    type: enum
    values: [None, Mild, Moderate, Severe]
```

#### **Relationship Types**
```yaml
  relationship_type:
    type: enum
    values: [sequel, prequel, spinoff, remake, reboot, adaptation, based_on, compilation, anthology]

  related_work_id:
    type: reference
    description: "Links to related media work"
```

#### **Discovery & Personalization**
```yaml
  watch_providers:
    type: array
    description: "Streaming services where available"

  similar_to:
    type: reference_array
    description: "AI-generated similarity links"

  user_tags:
    type: string_array
    description: "Community-generated tags"
```

---

## 3. Recommended Approach: Hybrid Strategy

### 3.1 Three-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: PUBLIC WEB LAYER (Schema.org)            │
│  - SEO-optimized JSON-LD                           │
│  - Movie, TVSeries, Person schemas                 │
│  - Google/Bing rich results                        │
└─────────────────────────────────────────────────────┘
                     ↑
                     │ Maps to
                     ↓
┌─────────────────────────────────────────────────────┐
│  Layer 2: PROFESSIONAL METADATA (EBUCore subset)   │
│  - Technical specs (resolution, codec, audio)      │
│  - Production metadata (crew, locations)           │
│  - Archive quality (preservation, rights)          │
└─────────────────────────────────────────────────────┘
                     ↑
                     │ Extends
                     ↓
┌─────────────────────────────────────────────────────┐
│  Layer 3: CUSTOM EXTENSIONS (Platform-specific)    │
│  - Genre/mood taxonomy                             │
│  - Content attributes (violence, language, etc.)   │
│  - Relationships (sequel, spinoff, remake)         │
│  - Discovery features (similar_to, watch_providers)│
│  - User engagement (preferences, lists, ratings)   │
└─────────────────────────────────────────────────────┘
```

### 3.2 Implementation Strategy

**Phase 1: Core Schema (Months 1-2)**
- Implement Schema.org Movie/TVSeries as base
- Add custom genre/mood taxonomy
- Implement content attribute system
- Create relationship graph structure

**Phase 2: Professional Metadata (Months 3-4)**
- Integrate EBUCore technical metadata
- Add crew role taxonomy
- Implement production metadata
- Support for multiple audio/video tracks

**Phase 3: Identifiers & Interoperability (Month 5)**
- Map to EIDR, ISAN, IMDb IDs
- Implement Wikidata sync for data enrichment
- Create import/export for common formats
- Build migration tools from existing systems

**Phase 4: Discovery & Personalization (Month 6)**
- User preference modeling (TV-Anytime inspired)
- Similarity algorithms
- Watch provider integration
- Community tagging system

---

### 3.3 Recommended Core Properties

```json
{
  "@context": "https://schema.org",
  "@type": "Movie",
  "identifier": [
    {"@type": "PropertyValue", "propertyID": "internal_id", "value": "12345"},
    {"@type": "PropertyValue", "propertyID": "EIDR", "value": "10.5240/XXXX"},
    {"@type": "PropertyValue", "propertyID": "IMDB", "value": "tt0111161"}
  ],
  "name": "The Shawshank Redemption",
  "alternateName": ["Rita Hayworth and Shawshank Redemption"],
  "datePublished": "1994-09-23",
  "duration": "PT142M",
  "inLanguage": ["en"],

  "genre": ["Drama", "Crime"],
  "genreSecondary": ["Prison Drama", "Redemption Story"],
  "moodTags": ["Uplifting", "Thought-Provoking", "Heartwarming"],
  "themes": ["Hope", "Friendship", "Redemption", "Injustice"],

  "contentRating": "R",
  "contentAttributes": {
    "violence": "Moderate",
    "language": "Moderate",
    "sexual": "Mild",
    "frightening": "Mild"
  },

  "director": {"@type": "Person", "name": "Frank Darabont"},
  "actor": [
    {"@type": "Person", "name": "Tim Robbins", "characterName": "Andy Dufresne"},
    {"@type": "Person", "name": "Morgan Freeman", "characterName": "Ellis Boyd 'Red' Redding"}
  ],
  "author": [{"@type": "Person", "name": "Stephen King"}],

  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": 9.3,
    "ratingCount": 2500000,
    "bestRating": 10
  },

  "relatedTo": [
    {
      "@type": "CreativeWork",
      "relationshipType": "based_on",
      "identifier": "stephen-king-rita-hayworth"
    }
  ],

  "watchProviders": ["Netflix", "Amazon Prime Video", "HBO Max"],

  "technicalMetadata": {
    "@context": "http://www.ebu.ch/metadata/ontologies/ebucore/ebucore#",
    "videoFormat": "1080p",
    "aspectRatio": "1.85:1",
    "audioFormat": "Dolby Digital 5.1"
  }
}
```

---

## 4. Licensing & Legal Considerations

| Standard | License Type | Commercial Use | Attribution | Modifications | Notes |
|----------|-------------|----------------|-------------|---------------|-------|
| Schema.org | CC BY-SA 3.0 | ✅ Yes | Required | Allowed | Share-alike required |
| EBUCore | EBU Public | ✅ Yes | Recommended | Allowed | Free and open |
| EIDR | Proprietary | ⚠️ Paid | N/A | No | Registry fees apply |
| ISAN | Proprietary | ⚠️ Paid | N/A | No | Registry fees apply |
| MovieLabs | Open | ✅ Yes | Recommended | Allowed | Free and open |
| W3C Media Ont | W3C License | ✅ Yes | Recommended | Allowed | Free and open |
| TV-Anytime | ETSI | ✅ Yes | Recommended | Allowed | Free and open |
| Wikidata | CC0 | ✅ Yes | None | Allowed | Public domain |
| Dublin Core | Open | ✅ Yes | None | Allowed | Free and open |
| IMDb Data | Proprietary | ⚠️ Licensed | N/A | No | Free non-commercial TSV, paid commercial |

**Recommendation:** All recommended standards (Schema.org, EBUCore, W3C, Wikidata) are free for commercial use. EIDR/ISAN optional as external identifiers only.

---

## 5. Data Sources for Seeding

### 5.1 Free & Open Sources
1. **Wikidata** (CC0) - 100K+ movies, multilingual, SPARQL API
2. **IMDb Non-Commercial Datasets** - TSV exports updated daily
3. **The Movie Database (TMDb)** - Free API (attribution required), 500K+ movies
4. **Open Movie Database (OMDb)** - Free API tier, IMDb data wrapper

### 5.2 Commercial Sources
1. **IMDb Licensed Data** (AWS Data Exchange) - 8M titles, 11M people, $$$
2. **Gracenote** (Nielsen) - Professional metadata, $$$
3. **Rovi / TiVo** - TV metadata specialist, $$$

### 5.3 Community Sources
1. **AniDB** - Anime database, free API
2. **MyAnimeList** - Anime/manga, unofficial API
3. **TheTVDB** - TV series focus, free API (registration)
4. **MusicBrainz** - Music metadata (future phase)

**Recommendation:** Start with Wikidata + TMDb for initial seeding, migrate to IMDb licensed data when budget allows.

---

## 6. Implementation Recommendations

### 6.1 Database Schema
```sql
-- Core tables (simplified example)
CREATE TABLE media_works (
  id UUID PRIMARY KEY,
  type VARCHAR(20), -- movie, tv_series, episode, etc.
  title JSONB, -- {"en": "Title", "ja": "タイトル"}
  release_date DATE,
  duration_minutes INT,
  schema_org JSONB, -- Full Schema.org representation
  ebucore_metadata JSONB, -- EBUCore technical metadata
  custom_metadata JSONB -- Custom extensions
);

CREATE TABLE external_identifiers (
  media_work_id UUID REFERENCES media_works(id),
  system VARCHAR(50), -- 'eidr', 'isan', 'imdb', 'tmdb', 'wikidata'
  identifier VARCHAR(255),
  PRIMARY KEY (media_work_id, system)
);

CREATE TABLE genres (
  id UUID PRIMARY KEY,
  name VARCHAR(100),
  parent_id UUID REFERENCES genres(id),
  taxonomy_source VARCHAR(50) -- 'schema.org', 'ebucore', 'custom'
);

CREATE TABLE media_genres (
  media_work_id UUID REFERENCES media_works(id),
  genre_id UUID REFERENCES genres(id),
  is_primary BOOLEAN,
  PRIMARY KEY (media_work_id, genre_id)
);

CREATE TABLE relationships (
  source_work_id UUID REFERENCES media_works(id),
  target_work_id UUID REFERENCES media_works(id),
  relationship_type VARCHAR(50), -- sequel, prequel, spinoff, etc.
  PRIMARY KEY (source_work_id, target_work_id, relationship_type)
);
```

### 6.2 API Design
```
GET /api/v1/media/{id}
  ?include=credits,ratings,relationships,technical
  &format=schema.org|ebucore|json

Response formats:
- JSON-LD (Schema.org default)
- EBUCore RDF/XML
- Custom JSON
- GraphQL support
```

### 6.3 Mapping Strategy
Create bidirectional mappings between standards:
```yaml
mappings:
  schema.org.Movie.director → ebucore.hasCreator (role: director)
  ebucore.hasAspectRatio → custom.technicalMetadata.aspectRatio
  imdb.genres → custom.genrePrimary (mapped taxonomy)
  wikidata.P57 (director) → schema.org.director
```

---

## 7. Risks & Challenges

### 7.1 Identified Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **Schema drift** - Standards evolve differently | Medium | High | Version all schemas, implement adapters |
| **Mapping inconsistencies** - Genre taxonomies conflict | High | Medium | Create authoritative taxonomy with mappings |
| **License compliance** - Misuse of proprietary data | Low | High | Strict license tracking, legal review |
| **Data quality** - Community sources have errors | High | Medium | Implement validation, human review for key titles |
| **Scalability** - JSONB fields grow unbounded | Medium | Medium | Implement field limits, archival strategy |
| **Breaking changes** - Schema.org updates break API | Low | High | Version API, gradual migrations |

### 7.2 Technical Challenges

1. **Genre normalization** - IMDb has ~20 genres, Wikidata has 100+, EBUCore has different set
   - **Solution:** Create master taxonomy with mappings to all sources

2. **Multilingual support** - Different standards handle i18n differently
   - **Solution:** JSONB fields with language keys `{"en": "...", "ja": "..."}`

3. **Relationship modeling** - Graph vs relational tradeoffs
   - **Solution:** Hybrid approach - PostgreSQL with relationship tables + optional graph DB

4. **Version control** - Metadata changes over time
   - **Solution:** Temporal tables or audit log for all changes

---

## 8. Conclusion & Next Steps

### 8.1 Final Recommendation

**ADOPT HYBRID APPROACH:**

1. **Schema.org** as public API foundation (SEO, interoperability)
2. **EBUCore subset** for professional/technical metadata
3. **Custom extensions** for unique platform features:
   - Genre/mood taxonomy
   - Content attributes
   - Relationship types
   - Discovery features

### 8.2 Immediate Next Steps

1. ✅ **Define Genre Taxonomy** - Map IMDb, Wikidata, EBUCore genres to master list
2. ✅ **Create JSON Schema** - Formalize the hybrid ontology with validation
3. ✅ **Build Import Pipeline** - Wikidata → TMDb → Custom format
4. ✅ **Prototype API** - Implement `/media/{id}` with Schema.org JSON-LD
5. ✅ **Seed Database** - Import 10K titles for testing

### 8.3 Long-Term Goals

- **Month 6:** Full EIDR integration for professional clients
- **Month 9:** Music metadata (MusicBrainz integration)
- **Month 12:** Podcast/audiobook expansion
- **Month 18:** Game metadata integration

---

## 9. Sources & References

### Standards Documentation
- [Ontology for Media Resources 1.0 - W3C](https://www.w3.org/TR/mediaont-10/)
- [MovieLabs Ontology for Media Creation v2.6+](https://movielabs.com/ontology-for-media-creation-2/)
- [EBU Core Metadata Set (EBUCore) - Tech 3293](https://tech.ebu.ch/metadata/ebucore)
- [Schema.org Movie Type](https://schema.org/Movie)
- [Schema.org TVSeries Type](https://schema.org/TVSeries)
- [EIDR Standards & Interoperability](https://www.eidr.org/standards-and-interoperability/)
- [ISAN & EIDR Metadata Schema Mapping](https://www.isan.org/docs/ISAN-EIDR_Metadata_Schema_Mapping.pdf)
- [TV-Anytime Broadcast Metadata Standard](https://dvb.org/?standard=broadcast-and-on-line-services-search-select-and-rightful-use-of-content-tv-anytime-part-3-metadata-sub-part-1-phase-1-metadata-schemas)

### Industry Resources
- [SMPTE, MovieLabs, EBU: Media in the Cloud Ontology Guide](https://www.smpte.org/blog/smpte-movielabs-and-ebu-publish-media-in-the-cloud-ontology-and-semantic-web-technology-navigation-guide)
- [How Metadata Standards Benefit the Media Industry - Accurity](https://www.accurity.ai/blog/how-metadata-standards-and-interoperability-benefit-the-media-industry/)
- [EIDR, ISAN Registry Updates - SmartContent](https://www.smartcontentonline.org/post/?u=2022/07/14/eidr-isan-provide-updates-on-their-id-standard-registries-2)

### Academic & Technical Papers
- [Creating a Meaningful Genre Schema using IMDb Data - Digital Humanities 2020](https://dh2020.adho.org/wp-content/uploads/2020/07/307_CreatingaMeaningfulGenreSchemaandMetadatausingIMDbdataforaLargeScaleDigitalHumanitiesProjectinMediaStudies.html)
- [IMDb Database Schema Documentation](https://www.researchgate.net/figure/Complete-schema-of-the-IMDb-database-with-8-main-relations-movie-person-genre_fig1_348079657)
- [AWS: How to Use IMDb Data in ML Applications](https://aws.amazon.com/blogs/media/how-to-use-imdb-data-in-search-and-machine-learning-applications/)

### Community Resources
- [Wikidata WikiProject Movies - Properties](https://www.wikidata.org/wiki/Wikidata:WikiProject_Movies/Properties)
- [W3C WebSchemas SchemaDotOrgTV Wiki](https://www.w3.org/wiki/WebSchemas/SchemaDotOrgTV)
- [Dublin Core Metadata Initiative - Media Arts](https://www.dublincore.org/conferences/2022/sessions/panel-metadata-for-visual-media-arts/)
- [Media APIs/Metadata Formats - W3C Web and TV IG](https://www.w3.org/2011/webtv/wiki/Media_APIs/Metadata_Formats)

### Data Sources
- [IMDb Non-Commercial Datasets](https://www.imdb.com/interfaces/)
- [Wikidata SPARQL Endpoint](https://www.wikidata.org/)
- [The Movie Database (TMDb) API](https://www.themoviedb.org/)
- [MusicBrainz Database](https://musicbrainz.org/)

---

**Report compiled:** 2025-12-06
**Research agent:** Claude Sonnet 4.5
**Status:** Ready for stakeholder review
