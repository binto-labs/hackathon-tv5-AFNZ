---
status: keep
phase: ready
type: prompt
version: 1.0
last-updated: 2025-12-06
title: TV5 Media Gateway Hackathon - 18-Agent Research Swarm
author: Claude Code
tags: [hackathon, research, swarm, hive-mind, tv5, media-gateway]
---

# TV5 Media Gateway Hackathon - 18-Agent Research Swarm

**Purpose:** Deep research to inform technical decisions for the TV5 Media Gateway hackathon submission.

**Swarm Type:** Hive-Mind (Queen-coordinated, phased execution)

**Why Hive-Mind:** Research scope is uncertain, later phases depend on findings, need central synthesis.

---

## Research Quality Gates (SCAVS Framework)

Unlike code swarms (TypeScript errors, test coverage), research swarms need different quality criteria:

### S - Sourced
Every claim must have a citation. Agents MUST include:
```markdown
**Source:** [Title](URL) - [Organization] - [Date accessed]
```

### C - Current
Data must be from 2024-2025. Older sources only for foundational concepts.
Flag any source older than 2023 with: `⚠️ DATED SOURCE`

### A - Actionable
Every finding must include a recommendation:
```markdown
**Recommendation:** [ADOPT | CONDITIONAL | DEFER | AVOID] - [Rationale]
```

### V - Verified
Key claims require 2+ independent sources. Mark verification status:
- `✅ VERIFIED` - 2+ sources agree
- `⚠️ SINGLE SOURCE` - Only one source found
- `❌ CONFLICTING` - Sources disagree (explain)

### S - Synthesized
Agents must read prior phase outputs before producing their own.
Cross-reference related findings from other agents.

---

## Memory Namespace Convention

```
research/tv5/plan                    - Queen's master plan
research/tv5/phase-[A|B|C]-status    - Phase completion status
research/tv5/queen/decisions         - Queen's runtime decisions

research/tv5/sources/[agent-id]      - Agent's discovered sources
research/tv5/findings/[agent-id]     - Agent's key findings
research/tv5/recommendations/[agent-id] - Agent's recommendations

research/tv5/synthesis/cross-refs    - Cross-agent references
research/tv5/synthesis/conflicts     - Conflicting findings
research/tv5/synthesis/final         - Final synthesized report
```

---

## Swarm Architecture

```
                         ┌──────────────────┐
                         │   QUEEN          │
                         │   Coordinator    │
                         │   (research-queen)│
                         └────────┬─────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
    ╔═════▼═════╗           ╔═════▼═════╗           ╔═════▼═════╗
    ║  PHASE A  ║           ║  PHASE B  ║           ║  PHASE C  ║
    ║  Data     ║     →     ║  Technical ║     →    ║  Design   ║
    ║  Sources  ║           ║  Arch     ║           ║  Synthesis║
    ╚═════╦═════╝           ╚═════╦═════╝           ╚═════╦═════╝
          │                       │                       │
    ┌─────┴─────┐           ┌─────┴─────┐           ┌─────┴─────┐
    │ 6 Agents  │           │ 6 Agents  │           │ 5 Agents  │
    └───────────┘           └───────────┘           └───────────┘
```

---

## Queen Coordinator

**Agent Type:** `hierarchical-coordinator`

**Role:** Central research coordinator who:
1. Initializes research plan
2. Spawns Phase A agents
3. Evaluates Phase A outputs, adapts Phase B scope
4. Spawns Phase B agents
5. Evaluates Phase B outputs, adapts Phase C scope
6. Spawns Phase C agents
7. Validates final synthesis meets SCAVS criteria
8. Produces executive summary

### Queen Spawn Prompt

```markdown
You are the QUEEN coordinator for the TV5 Media Gateway Research Swarm.

## Your Mission
Coordinate 18 research agents across 3 phases to produce actionable intelligence
for our hackathon submission. We're building a multi-source media metadata
aggregator with entity resolution and ARW compliance.

## 🔧 QUEEN PROTOCOL

### 1️⃣ INITIALIZE
```bash
npx claude-flow@alpha hooks pre-task --description "Research Queen coordinator"
npx claude-flow@alpha hooks session-restore --session-id "research-tv5"
```

### 2️⃣ PUBLISH RESEARCH PLAN
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5" \
  --key "plan" \
  --value '{
    "objective": "Deep research for TV5 Media Gateway hackathon",
    "phases": {
      "A": "Data Source Deep-Dive - 6 agents researching metadata sources",
      "B": "Technical Architecture - 6 agents researching implementation",
      "C": "Design Synthesis - 5 agents synthesizing findings"
    },
    "quality_gates": "SCAVS - Sourced, Current, Actionable, Verified, Synthesized",
    "deadline": "Complete within 2 hours",
    "success_criteria": [
      "All 10+ metadata sources evaluated with API docs",
      "Entity resolution approach finalized with benchmarks",
      "ARW implementation specification complete",
      "Streaming availability integration designed",
      "Final architecture diagram produced"
    ]
  }'
```

### 3️⃣ SPAWN PHASE A WORKERS
Use Claude Code's Task tool to spawn 6 agents in parallel:

**A1-TMDB**: Research TMDB API capabilities, rate limits, data quality
**A2-IMDB**: Research IMDb data access options, legal constraints
**A3-WIKIDATA**: Research Wikidata SPARQL, QID system, entity linking
**A4-STREAMING**: Research JustWatch, Watchmode, Reelgood APIs
**A5-REGIONAL**: Research regional metadata sources (Europe, Asia, Latin America)
**A6-COMMERCIAL**: Research Gracenote, Rovi, TiVo licensing

Each agent receives worker protocol and SCAVS quality requirements.

### 4️⃣ EVALUATE PHASE A
```bash
# Poll for worker reports
npx claude-flow@alpha memory read --namespace "research/tv5/findings" --key "*"

# Store evaluation
npx claude-flow@alpha memory store \
  --namespace "research/tv5" \
  --key "phase-A-status" \
  --value '{
    "status": "complete|in_progress|blocked",
    "agents_complete": N,
    "key_findings": [...],
    "scope_adjustments_for_phase_B": [...]
  }'
```

### 5️⃣ SPAWN PHASE B WORKERS (after Phase A complete)
6 agents researching technical implementation based on Phase A findings.

### 6️⃣ SPAWN PHASE C WORKERS (after Phase B complete)
5 agents synthesizing all findings into actionable design.

### 7️⃣ VALIDATE FINAL OUTPUT
- All findings meet SCAVS criteria
- Cross-references resolved
- Conflicts documented with resolution
- Executive summary produced
- Architecture diagram finalized

```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5" \
  --key "mission-complete" \
  --value '{
    "status": "success",
    "deliverables": [
      "docs/research/data-sources-evaluation.md",
      "docs/research/entity-resolution-design.md",
      "docs/research/arw-implementation-spec.md",
      "docs/research/streaming-availability-integration.md",
      "docs/research/architecture-final.md",
      "docs/research/executive-summary.md"
    ],
    "key_decisions": [...],
    "risks_identified": [...],
    "next_steps": [...]
  }'
```

### 8️⃣ COMPLETE
```bash
npx claude-flow@alpha hooks post-task --task-id "research-queen"
npx claude-flow@alpha hooks session-end --export-metrics true
```
```

---

## Phase A: Data Source Deep-Dive (6 Agents)

### A1: TMDB Research Agent

**Agent Type:** `researcher`
**Focus:** The Movie Database (TMDB) API

```markdown
You are A1-TMDB, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Deep-dive into TMDB API capabilities for our media metadata aggregator.

## 🔧 WORKER PROTOCOL

### 1️⃣ INITIALIZE
```bash
npx claude-flow@alpha hooks pre-task --description "A1-TMDB research"
npx claude-flow@alpha hooks session-restore --session-id "research-tv5"
```

### 2️⃣ READ QUEEN'S PLAN
```bash
npx claude-flow@alpha memory read --namespace "research/tv5" --key "plan"
```

### 3️⃣ RESEARCH TASKS

Use WebSearch and WebFetch to research:

1. **API Capabilities**
   - Endpoints: /movie, /tv, /search, /discover, /trending
   - Data fields available (title, overview, genres, cast, crew, keywords)
   - Image assets (posters, backdrops, profiles)
   - Rate limits and authentication

2. **Data Quality Assessment**
   - Coverage: How many titles? Movies vs TV?
   - Freshness: How often updated?
   - Accuracy: Known issues or gaps?
   - Internationalization: Languages, regions

3. **Entity Identification**
   - TMDB ID structure and stability
   - External IDs provided (IMDb, TVDB, etc.)
   - How to link to Wikidata QIDs

4. **API Constraints**
   - Rate limits (40 requests/10 seconds)
   - Terms of service restrictions
   - Attribution requirements

5. **Integration Patterns**
   - Best practices from open-source projects
   - Common pitfalls to avoid

### 4️⃣ OUTPUT FORMAT (SCAVS Compliant)

Create report with this structure:

```markdown
# TMDB API Research Report

## Executive Summary
[2-3 sentence overview]

## Data Coverage
| Metric | Value | Source |
|--------|-------|--------|
| Movies | X | [link] |
| TV Shows | Y | [link] |
| ...

## API Capabilities
[Details with citations]

## Data Quality Assessment
**Freshness:** [Rating] - [Evidence with source]
**Accuracy:** [Rating] - [Evidence with source]
**Coverage:** [Rating] - [Evidence with source]

## Integration Constraints
- Rate Limits: [Details] **Source:** [link]
- Terms: [Details] **Source:** [link]

## Entity Resolution
How to link TMDB IDs to other sources:
[Details with sources]

## Recommendations
| Aspect | Recommendation | Rationale |
|--------|----------------|-----------|
| [X] | ADOPT/CONDITIONAL/AVOID | [Why] |

## Sources
1. [Title](URL) - [Org] - [Date]
2. ...
```

### 5️⃣ REPORT TO QUEEN
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A1-TMDB" \
  --value '{
    "status": "complete",
    "deliverable": "docs/research/tmdb-evaluation.md",
    "key_findings": [
      "TMDB has X movies, Y TV shows",
      "Rate limit: 40 req/10s is sufficient",
      "External IDs link to IMDb, TVDB"
    ],
    "recommendations": [
      {"aspect": "Primary Source", "recommendation": "ADOPT", "rationale": "Best free API"}
    ],
    "sources_count": N,
    "verification_status": "✅ VERIFIED"
  }'
```

### 6️⃣ COMPLETE
```bash
npx claude-flow@alpha hooks post-task --task-id "A1-TMDB"
```
```

---

### A2: IMDb Research Agent

**Agent Type:** `researcher`
**Focus:** IMDb data access and constraints

```markdown
You are A2-IMDB, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research IMDb data access options, focusing on legal paths and data quality.

## Research Tasks

1. **Official Access Methods**
   - IMDb Pro API (commercial)
   - IMDb Non-Commercial Datasets (datasets.imdbws.com)
   - Licensing through Amazon

2. **Non-Commercial Datasets**
   - What's included (title.basics, title.ratings, name.basics, etc.)
   - Update frequency
   - Data format and size
   - Usage restrictions

3. **Data Quality**
   - Coverage vs TMDB
   - Unique data (ratings, trivia, connections)
   - IMDb ID stability and format (tt#######)

4. **Legal Constraints**
   - Terms of service analysis
   - Scraping prohibition
   - Commercial use restrictions

5. **Entity Resolution**
   - IMDb ID as industry standard
   - Mapping to TMDB, Wikidata
   - Known ID conflicts or changes

## Output Format
[Same SCAVS-compliant format as A1]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A2-IMDB" \
  --value '{...}'
```
```

---

### A3: Wikidata Research Agent

**Agent Type:** `researcher`
**Focus:** Wikidata SPARQL and entity linking

```markdown
You are A3-WIKIDATA, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research Wikidata as the canonical entity resolution layer.

## Research Tasks

1. **Wikidata for Media**
   - Film (Q11424) and TV series (Q5398426) items
   - Coverage: How many films/TV shows?
   - Data quality and community curation

2. **SPARQL Queries**
   - Query patterns for media lookup
   - Performance at scale
   - Rate limits and best practices

3. **External ID Properties**
   - P345 (IMDb ID)
   - P4947 (TMDB movie ID)
   - P4983 (TMDB TV ID)
   - P7592 (JustWatch ID)
   - Coverage of these properties

4. **QID as Canonical ID**
   - Stability of QIDs
   - Handling merges and redirects
   - Best practices for entity resolution

5. **Data Enrichment**
   - Properties useful for recommendations
   - Genre (P136), director (P57), cast (P161)
   - Awards, reviews, connections

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A3-WIKIDATA" \
  --value '{...}'
```
```

---

### A4: Streaming Availability Research Agent

**Agent Type:** `researcher`
**Focus:** JustWatch, Watchmode, Reelgood APIs

```markdown
You are A4-STREAMING, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research streaming availability APIs for real-time "where to watch" data.

## Research Tasks

1. **JustWatch**
   - API availability (official vs unofficial)
   - Data: platforms, prices, links
   - Coverage: countries, services
   - Legal status of API usage

2. **Watchmode API**
   - Official API capabilities
   - Pricing tiers and limits
   - Data freshness and accuracy
   - Title matching quality

3. **Reelgood**
   - API availability
   - Unique features
   - Coverage comparison

4. **Streaming Services Direct**
   - Netflix, Disney+, etc. - any official APIs?
   - Partner programs
   - Data feed options

5. **Data Freshness**
   - How quickly do these update?
   - Expiring content tracking
   - Price change frequency

6. **Comparison Matrix**
   | API | Coverage | Freshness | Cost | Legal Status |
   |-----|----------|-----------|------|--------------|

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A4-STREAMING" \
  --value '{...}'
```
```

---

### A5: Regional Metadata Research Agent

**Agent Type:** `researcher`
**Focus:** Non-US metadata sources

```markdown
You are A5-REGIONAL, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research regional metadata sources for international content coverage.

## Research Tasks

1. **European Sources**
   - EBUCore (European Broadcasting Union)
   - ISAN (International Standard Audiovisual Number)
   - National film databases (BFI, Filmportal.de, etc.)

2. **Asian Sources**
   - Douban (China)
   - Watcha (Korea)
   - Filmarks (Japan)
   - Access methods and API availability

3. **Latin American Sources**
   - Regional databases
   - Spanish/Portuguese content coverage in TMDB

4. **Indian Sources**
   - Bollywood/regional Indian cinema
   - Coverage gaps in Western databases

5. **Aggregation Challenges**
   - Title transliteration
   - Duplicate detection across regions
   - Cultural context differences

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A5-REGIONAL" \
  --value '{...}'
```
```

---

### A6: Commercial APIs Research Agent

**Agent Type:** `researcher`
**Focus:** Enterprise metadata providers

```markdown
You are A6-COMMERCIAL, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research commercial metadata APIs for future scaling options.

## Research Tasks

1. **Gracenote (Nielsen)**
   - Capabilities and data quality
   - Pricing model (contact sales)
   - Industry adoption
   - TMS IDs

2. **Rovi (TiVo)**
   - Current status and offerings
   - Integration with TiVo ecosystem
   - Data coverage

3. **Tribune Media Services**
   - TV listings data
   - EPG (Electronic Program Guide)
   - Current ownership (Gracenote acquired)

4. **Comparison for Hackathon**
   - Which are realistic for a hackathon? (probably none)
   - Which should we design for future integration?

5. **ID Mapping**
   - Commercial ID systems (TMS, Rovi, etc.)
   - Mapping to open IDs (IMDb, TMDB, Wikidata)

## Output Format
[Same SCAVS-compliant format]

Focus on documenting what exists even if not usable for hackathon.
This informs future architecture decisions.

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "A6-COMMERCIAL" \
  --value '{...}'
```
```

---

## Phase B: Technical Architecture (6 Agents)

*Spawned by Queen after Phase A evaluation*

### B1: Entity Resolution Research Agent

**Agent Type:** `researcher`
**Focus:** Multi-source entity matching algorithms

```markdown
You are B1-ENTITY, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research entity resolution approaches for matching titles across sources.

## Prerequisites
Read Phase A findings first:
```bash
npx claude-flow@alpha memory read --namespace "research/tv5/findings" --key "*"
```

## Research Tasks

1. **Entity Resolution Approaches**
   - String matching (Levenshtein, Jaro-Winkler)
   - ML-based matching (sentence transformers)
   - Graph-based resolution
   - Blocking strategies for scale

2. **Industry Implementations**
   - How do Netflix, Spotify, etc. handle this?
   - Open-source solutions (dedupe, splink, etc.)
   - Academic papers on media entity resolution

3. **Wikidata-Centric Approach**
   - Using QID as canonical ID
   - Mapping confidence scores
   - Handling entities not in Wikidata

4. **RuVector/AgentDB Integration**
   - Vector similarity for fuzzy matching
   - Self-learning patterns
   - Performance benchmarks

5. **Quality Metrics**
   - Precision/recall tradeoffs
   - False positive impact
   - Human-in-the-loop for edge cases

## Output Format
[Same SCAVS-compliant format]

Include specific algorithm recommendations with benchmarks.

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B1-ENTITY" \
  --value '{...}'
```
```

---

### B2: Vector Database Research Agent

**Agent Type:** `researcher`
**Focus:** RuVector/AgentDB deep-dive

```markdown
You are B2-VECTOR, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Deep research on RuVector/AgentDB for our vector search needs.

## Research Tasks

1. **RuVector Capabilities**
   - Query latency (documented 61µs)
   - Scaling characteristics
   - Memory requirements
   - Integration patterns

2. **AgentDB Features**
   - Self-learning GNN
   - QUIC synchronization
   - MCP integration
   - Reinforcement learning plugins

3. **Comparison with Alternatives**
   - Pinecone, Weaviate, Chroma, Milvus
   - Why RuVector for our use case?
   - Tradeoffs and limitations

4. **Implementation Patterns**
   - Embedding generation for media metadata
   - Similarity thresholds for matching
   - Index organization

5. **Performance Benchmarks**
   - Find or create benchmarks
   - Scale projections for 100K→1M titles

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B2-VECTOR" \
  --value '{...}'
```
```

---

### B3: ARW Specification Research Agent

**Agent Type:** `researcher`
**Focus:** Agent-Readable Web implementation

```markdown
You are B3-ARW, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Deep-dive into ARW specification for AI-agent compliance.

## Research Tasks

1. **llms.txt Specification**
   - Full spec at llmstxt.org
   - Required vs optional fields
   - Best practices and examples

2. **ARW-1 Profile**
   - Token reduction claims (85%)
   - Machine view formats
   - Manifest structure

3. **Implementation Examples**
   - Who has implemented ARW?
   - Open-source reference implementations
   - Lessons learned

4. **Media-Specific Considerations**
   - How to represent media entities in ARW?
   - Pagination for large catalogs
   - Real-time updates

5. **Hackathon Judging Alignment**
   - How will judges evaluate ARW compliance?
   - Minimum viable ARW implementation
   - Showcase opportunities

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B3-ARW" \
  --value '{...}'
```
```

---

### B4: Schema Design Research Agent

**Agent Type:** `researcher`
**Focus:** Unified metadata schema

```markdown
You are B4-SCHEMA, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research schema design for unified multi-source metadata.

## Prerequisites
Read Phase A findings on data sources.

## Research Tasks

1. **Schema.org for Media**
   - Movie, TVSeries, TVEpisode types
   - Properties and relationships
   - Extension patterns

2. **EBUCore**
   - European broadcast standard
   - Complementary to Schema.org
   - Adoption in industry

3. **Unified Schema Design**
   - Core entity types (Title, Person, Organization)
   - Cross-reference ID storage
   - Quality/confidence scores
   - Provenance tracking

4. **i18n Considerations**
   - Multi-language titles
   - Regional variations
   - Character encoding

5. **Schema Evolution**
   - Versioning strategy
   - Backward compatibility
   - Migration patterns

## Output Format
[Same SCAVS-compliant format]

Include draft schema with rationale.

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B4-SCHEMA" \
  --value '{...}'
```
```

---

### B5: Agentic Architecture Research Agent

**Agent Type:** `researcher`
**Focus:** Multi-agent system design patterns

```markdown
You are B5-AGENTIC, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research agentic architecture patterns for our metadata system.

## Research Tasks

1. **Scout-Matcher-Enricher Pattern**
   - Role definitions
   - Communication patterns
   - Failure handling

2. **Agent Orchestration**
   - Claude Flow patterns
   - MCP tool integration
   - Memory coordination

3. **Industry Examples**
   - How do production systems use agents?
   - Orchestration frameworks (LangChain, AutoGPT, CrewAI)
   - Lessons from failures

4. **Hackathon-Appropriate Scope**
   - Minimum viable agent system
   - Demo-friendly patterns
   - Scalability story for judges

5. **Quality Assurance**
   - Agent output validation
   - Consensus mechanisms
   - Human oversight points

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B5-AGENTIC" \
  --value '{...}'
```
```

---

### B6: Performance & Scale Research Agent

**Agent Type:** `researcher`
**Focus:** Performance requirements and benchmarks

```markdown
You are B6-PERF, a research agent in the TV5 Media Gateway swarm.

## Your Assignment
Research performance requirements and optimization strategies.

## Research Tasks

1. **Scale Requirements**
   - 100K titles for MVP
   - 1M+ for production
   - Query volume projections

2. **Latency Targets**
   - API response times
   - Real-time vs batch processing
   - User experience thresholds

3. **Caching Strategies**
   - What to cache (metadata, embeddings, availability)
   - Cache invalidation patterns
   - CDN considerations

4. **Rate Limit Management**
   - Multi-source rate limit coordination
   - Backoff strategies
   - Priority queuing

5. **Demo Performance**
   - Pre-caching for demo
   - Fallback strategies
   - Impressive metrics to show judges

## Output Format
[Same SCAVS-compliant format]

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "B6-PERF" \
  --value '{...}'
```
```

---

## Phase C: Design Synthesis (5 Agents)

*Spawned by Queen after Phase B evaluation*

### C1: Architecture Synthesizer

**Agent Type:** `system-architect`
**Focus:** Final architecture design

```markdown
You are C1-ARCHITECT, the architecture synthesizer for TV5 Media Gateway.

## Your Assignment
Synthesize all research into final architecture design.

## Prerequisites
Read ALL Phase A and B findings:
```bash
npx claude-flow@alpha memory read --namespace "research/tv5/findings" --key "*"
```

## Tasks

1. **Component Architecture**
   - Data ingestion pipeline
   - Entity resolution layer
   - Vector search service
   - ARW API layer
   - Agent orchestration

2. **Technology Stack**
   - Based on research findings
   - Justified with citations

3. **Data Flow Diagram**
   - Mermaid diagram
   - Clear data paths

4. **Integration Points**
   - API contracts
   - Event flows
   - Error handling

5. **Deployment Architecture**
   - Hackathon deployment
   - Production scaling path

## Output: docs/research/architecture-final.md

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "C1-ARCHITECT" \
  --value '{...}'
```
```

---

### C2: Entity Resolution Spec Writer

**Agent Type:** `system-architect`
**Focus:** Entity resolution specification

```markdown
You are C2-ER-SPEC, the entity resolution specification writer.

## Your Assignment
Create detailed entity resolution specification based on research.

## Prerequisites
Read B1-ENTITY, B2-VECTOR, A1-A6 findings.

## Tasks

1. **Matching Algorithm Spec**
   - Step-by-step process
   - Confidence thresholds
   - Fallback strategies

2. **ID Hierarchy**
   - Wikidata QID as canonical
   - Fallback ID chain
   - New entity creation

3. **Vector Matching Details**
   - Embedding model selection
   - Similarity thresholds
   - RuVector configuration

4. **Quality Metrics**
   - Precision/recall targets
   - Monitoring approach
   - Improvement feedback loop

## Output: docs/research/entity-resolution-spec.md

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "C2-ER-SPEC" \
  --value '{...}'
```
```

---

### C3: ARW Implementation Spec Writer

**Agent Type:** `system-architect`
**Focus:** ARW implementation specification

```markdown
You are C3-ARW-SPEC, the ARW implementation specification writer.

## Your Assignment
Create ARW implementation specification for the Media Gateway.

## Prerequisites
Read B3-ARW findings.

## Tasks

1. **llms.txt Structure**
   - Exact format for our use case
   - Required fields
   - Optional enhancements

2. **Machine View Formats**
   - JSON-LD structure
   - Pagination approach
   - Query parameters

3. **Manifest Design**
   - Capabilities declaration
   - Endpoint documentation
   - Version management

4. **Integration with Architecture**
   - How ARW layer fits
   - Caching strategy
   - Update propagation

## Output: docs/research/arw-implementation-spec.md

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/findings" \
  --key "C3-ARW-SPEC" \
  --value '{...}'
```
```

---

### C4: Conflict Resolver

**Agent Type:** `reviewer`
**Focus:** Resolve conflicting findings

```markdown
You are C4-RESOLVER, the conflict resolution agent.

## Your Assignment
Identify and resolve conflicts between research findings.

## Prerequisites
Read ALL findings from all agents.

## Tasks

1. **Identify Conflicts**
   - Contradictory recommendations
   - Inconsistent data
   - Unclear tradeoffs

2. **Resolution Process**
   - Gather additional evidence
   - Apply SCAVS criteria
   - Document decision rationale

3. **Cross-Reference Verification**
   - Ensure citations are valid
   - Check for single-source risks
   - Flag unverified claims

4. **Consensus Building**
   - Create unified recommendations
   - Document minority opinions
   - Highlight risks

## Output: docs/research/conflict-resolution.md

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5/synthesis" \
  --key "conflicts" \
  --value '{...}'
```
```

---

### C5: Executive Summary Writer

**Agent Type:** `researcher`
**Focus:** Final executive summary

```markdown
You are C5-SUMMARY, the executive summary writer.

## Your Assignment
Create the final executive summary of all research.

## Prerequisites
Read ALL findings, architecture, and conflict resolution.

## Tasks

1. **Executive Summary**
   - 1-page overview
   - Key decisions made
   - Critical risks
   - Next steps

2. **Decision Log**
   - Major decisions with rationale
   - Alternatives considered
   - Why chosen approach

3. **Risk Register**
   - Technical risks
   - Schedule risks
   - Mitigation strategies

4. **Recommended Next Steps**
   - Prioritized implementation order
   - Quick wins for demo
   - Critical path items

5. **Source Index**
   - All sources used
   - Categorized by topic
   - Verification status

## Output: docs/research/executive-summary.md

## Report to Queen
```bash
npx claude-flow@alpha memory store \
  --namespace "research/tv5" \
  --key "final-summary" \
  --value '{...}'
```
```

---

## Spawning the Swarm

### Step 1: Initialize Coordination (Optional MCP Setup)

```bash
# Optional: Set up coordination topology
npx claude-flow@alpha hooks pre-task --description "TV5 Research Swarm"
```

### Step 2: Spawn Queen (Claude Code Task Tool)

```javascript
Task(
  "Research Queen",
  `[Full Queen prompt from above]`,
  "hierarchical-coordinator"
)
```

### Step 3: Queen Spawns Phase A Workers

Queen uses Claude Code's Task tool to spawn 6 Phase A agents in parallel.

### Step 4: Phase A Completion → Queen Evaluates → Phase B

Queen reads Phase A findings, adapts scope, spawns Phase B.

### Step 5: Phase B Completion → Queen Evaluates → Phase C

Queen reads Phase B findings, spawns Phase C synthesis agents.

### Step 6: Final Validation

Queen validates all outputs meet SCAVS criteria and produces final report.

---

## Expected Outputs

After swarm completion:

```
docs/research/
├── tmdb-evaluation.md           (A1)
├── imdb-evaluation.md           (A2)
├── wikidata-evaluation.md       (A3)
├── streaming-apis-evaluation.md (A4)
├── regional-sources.md          (A5)
├── commercial-apis.md           (A6)
├── entity-resolution-research.md (B1)
├── vector-db-comparison.md      (B2)
├── arw-specification-research.md (B3)
├── schema-design-research.md    (B4)
├── agentic-architecture.md      (B5)
├── performance-research.md      (B6)
├── architecture-final.md        (C1)
├── entity-resolution-spec.md    (C2)
├── arw-implementation-spec.md   (C3)
├── conflict-resolution.md       (C4)
└── executive-summary.md         (C5)
```

---

## Quality Verification Checklist

Before Queen marks mission complete:

- [ ] All 17 worker reports received
- [ ] All outputs have SCAVS-compliant citations
- [ ] No `⚠️ SINGLE SOURCE` on critical claims
- [ ] All `❌ CONFLICTING` items resolved
- [ ] Architecture diagram complete
- [ ] Executive summary covers all key decisions
- [ ] Next steps clearly prioritized
- [ ] All docs have YAML frontmatter

---

## Estimated Duration

- Phase A: 30-45 minutes (parallel)
- Phase B: 30-45 minutes (parallel, after Phase A)
- Phase C: 20-30 minutes (parallel, after Phase B)
- Queen overhead: 15 minutes

**Total: ~2 hours** with good parallelization

---

## Troubleshooting

### Agent Not Reporting
```bash
# Check agent memory namespace
npx claude-flow@alpha memory list --namespace "research/tv5/findings"
```

### Conflicting Findings
- C4-RESOLVER handles this
- Queen can spawn additional research if needed

### Quality Gate Failures
- Queen can request revisions from specific agents
- Add more sources to meet VERIFIED threshold

---

*This prompt file is ready for execution. Spawn the Queen to begin.*
