# RuVector Quick Start Guide for Media Discovery Platform

**Purpose**: Get RuVector (via AgentDB) running for semantic media search in <5 minutes

---

## Installation

```bash
# Option 1: Via Claude Flow (RECOMMENDED for hackathon)
npx claude-flow@alpha init --force

# Option 2: Standalone AgentDB
npm install -g agentdb
npx agentdb init

# Option 3: Direct RuVector
npm install ruvector
```

---

## Basic Usage

### 1. Initialize Database

```javascript
// Via MCP (Claude Flow)
await mcp__claude_flow__agentdb_init({
  dimension: 384,  // For Sentence-BERT embeddings
  metric: 'cosine'
});

// Via Direct API
import { AgentDB } from 'agentdb';
const db = new AgentDB({ dimension: 384 });
```

### 2. Insert Media Embeddings

```javascript
// Single insert
await db.insert({
  id: 'inception-2010',
  vector: [0.12, 0.45, 0.78, ...], // 384-dimensional embedding
  metadata: {
    title: 'Inception',
    genre: 'sci-fi',
    mood: 'dark',
    rating: 8.8,
    year: 2010
  }
});

// Batch insert (500x faster)
await db.insertBatch([
  { id: 'inception-2010', vector: [...], metadata: {...} },
  { id: 'interstellar-2014', vector: [...], metadata: {...} },
  { id: 'tenet-2020', vector: [...], metadata: {...} }
]);
```

### 3. Semantic Search

```javascript
// Basic vector search
const results = await db.search({
  query: userQueryEmbedding,  // "find something like Inception but lighter"
  k: 10,
  threshold: 0.75
});

// With metadata filtering (Cypher)
const filtered = await db.query(`
  MATCH (media)
  WHERE media.genre = "sci-fi"
    AND media.mood IN ["light", "moderate"]
    AND media.rating > 7.0
  WITH media, vectorSimilarity(media.embedding, $query) AS score
  WHERE score > 0.75
  RETURN media
  ORDER BY score DESC
  LIMIT 10
`, { query: userQueryEmbedding });
```

### 4. Graph-Based Recommendations

```javascript
// Collaborative filtering
const recommendations = await db.query(`
  MATCH (user:User {id: $userId})-[:VIEWED]->(media)
  MATCH (media)-[:SIMILAR_TO]->(recommendation)
  WHERE NOT (user)-[:VIEWED]->(recommendation)
    AND recommendation.genre IN $preferredGenres
  RETURN recommendation
  ORDER BY recommendation.score DESC
  LIMIT 10
`, {
  userId: currentUser.id,
  preferredGenres: ['sci-fi', 'thriller']
});
```

---

## MCP Integration (Claude Flow)

### Available Tools

```javascript
// Initialize
mcp__claude_flow__agentdb_init()

// Insert data
mcp__claude_flow__agentdb_insert({ vector, metadata })
mcp__claude_flow__agentdb_insert_batch({ vectors: [...] })

// Search
mcp__claude_flow__agentdb_search({ query, k, filter })

// Pattern learning (self-improvement)
mcp__claude_flow__agentdb_pattern_store({ pattern, data })
mcp__claude_flow__agentdb_pattern_search({ pattern })

// Statistics
mcp__claude_flow__agentdb_stats()
```

### Example Workflow

```javascript
// 1. Store embeddings
await mcp__claude_flow__agentdb_insert_batch({
  vectors: mediaCatalog.map(media => ({
    vector: media.embedding,
    metadata: {
      id: media.id,
      title: media.title,
      genre: media.genre,
      mood: media.mood
    }
  }))
});

// 2. Search
const results = await mcp__claude_flow__agentdb_search({
  query: userQueryEmbedding,
  k: 10,
  filter: { genre: 'sci-fi', mood: 'light' }
});

// 3. Learn from interaction (automatic GNN improvement)
await mcp__claude_flow__agentdb_pattern_store({
  pattern: 'user-search-interaction',
  data: {
    userId: user.id,
    query: userQuery,
    results: results.map(r => r.id),
    selected: selectedMedia.id,
    timestamp: Date.now()
  }
});
```

---

## Performance Tips

### 1. Batch Operations

```javascript
// ❌ SLOW: Sequential inserts (1s for 100 items)
for (const media of catalog) {
  await db.insert(media);
}

// ✅ FAST: Batch insert (2ms for 100 items)
await db.insertBatch(catalog);
```

### 2. Use Appropriate Dimensions

```javascript
// ✅ Recommended: 384D (Sentence-BERT mini)
// - Fast: 61µs per query
// - Accurate: Good for most use cases
// - Efficient: 18% less memory

// ⚠️ Alternative: 768D (Sentence-BERT base)
// - Slower: ~120µs per query
// - More accurate: Better semantic understanding
// - More memory: 2x storage

// 🎯 For hackathon: Use 384D
```

### 3. Leverage Auto-Compression

```javascript
// RuVector automatically compresses based on access patterns
// No configuration needed!

// Hot data (recent/frequent): f32 (no compression)
// Warm data: f16 (2x compression)
// Cool data: PQ8 (8x compression)
// Archive: Binary (32x compression)
```

### 4. Enable Self-Learning

```javascript
// Track user interactions to improve search over time
await db.track({
  userId: user.id,
  mediaId: media.id,
  interaction: 'view' | 'like' | 'skip',
  timestamp: Date.now()
});

// GNN will automatically:
// - Strengthen paths to popular media
// - Learn genre/mood preferences
// - Optimize search topology
// Result: 98% prevention of performance degradation
```

---

## Embedding Generation

### Using Sentence-BERT (Recommended)

```bash
npm install @xenova/transformers
```

```javascript
import { pipeline } from '@xenova/transformers';

// Load model (384D embeddings)
const embedder = await pipeline(
  'feature-extraction',
  'Xenova/all-MiniLM-L6-v2'
);

// Generate embedding for media
const embedding = await embedder(media.title + ' ' + media.plot, {
  pooling: 'mean',
  normalize: true
});

// Store in RuVector
await db.insert({
  id: media.id,
  vector: Array.from(embedding.data),
  metadata: media
});
```

### Multi-Field Embeddings

```javascript
// Combine multiple fields for richer embeddings
async function generateMediaEmbedding(media) {
  const text = [
    media.title,
    media.plot,
    `Genre: ${media.genre}`,
    `Mood: ${media.mood}`,
    media.keywords.join(', ')
  ].join(' | ');

  const embedding = await embedder(text, {
    pooling: 'mean',
    normalize: true
  });

  return Array.from(embedding.data);
}
```

---

## Query Examples

### 1. Semantic Search: "Find something like Inception but lighter"

```javascript
// Step 1: Embed user query
const query = "sci-fi thriller with complex plot but lighter tone";
const queryEmbedding = await embedder(query);

// Step 2: Search with metadata filter
const results = await db.query(`
  MATCH (media)
  WHERE media.genre = "sci-fi"
    AND media.mood IN ["light", "moderate", "uplifting"]
  WITH media, vectorSimilarity(media.embedding, $query) AS score
  WHERE score > 0.7
  RETURN media
  ORDER BY score DESC
  LIMIT 10
`, { query: Array.from(queryEmbedding.data) });

// Results might include:
// - Arrival (sci-fi, thoughtful, optimistic)
// - Big Hero 6 (sci-fi, fun, light)
// - Edge of Tomorrow (sci-fi action, less dark)
```

### 2. Mood-Based Discovery

```javascript
// "I'm feeling anxious, recommend something calming"
const calmingMedia = await db.query(`
  MATCH (media)
  WHERE media.mood IN ["calm", "peaceful", "uplifting", "heartwarming"]
    AND media.pacing = "slow"
  RETURN media
  ORDER BY media.rating DESC
  LIMIT 10
`);
```

### 3. Collaborative Filtering

```javascript
// "Users who liked what I liked also enjoyed..."
const recommendations = await db.query(`
  MATCH (me:User {id: $myId})-[:LIKED]->(media)
  MATCH (other:User)-[:LIKED]->(media)
  MATCH (other)-[:LIKED]->(rec)
  WHERE NOT (me)-[:LIKED]->(rec)
    AND me <> other
  RETURN rec, COUNT(other) AS popularity
  ORDER BY popularity DESC
  LIMIT 10
`, { myId: currentUser.id });
```

### 4. Hybrid Search (Vector + BM25)

```javascript
// Combine semantic search with keyword matching
const hybrid = await db.hybridSearch({
  vector: queryEmbedding,
  keywords: ['space', 'time', 'mind-bending'],
  vectorWeight: 0.7,  // 70% semantic, 30% keyword
  keywordWeight: 0.3,
  k: 10
});
```

---

## Monitoring & Debugging

### Check Database Stats

```javascript
const stats = await db.stats();
console.log({
  vectorCount: stats.count,
  memoryUsage: stats.memory,
  avgQueryTime: stats.avgLatency,
  indexSize: stats.indexSize
});
```

### Performance Metrics

```javascript
// Via MCP
const metrics = await mcp__claude_flow__agentdb_stats();

// Direct API
const perf = await db.benchmark({
  queryCount: 100,
  k: 10
});
console.log({
  avgLatency: perf.avgLatency,  // Should be ~61µs
  p95Latency: perf.p95,
  throughput: perf.qps           // Should be ~16,400 QPS
});
```

### Debug Slow Queries

```javascript
// Enable query profiling
const results = await db.search({
  query: embedding,
  k: 10,
  debug: true
});

console.log({
  totalTime: results.timing.total,
  indexSearch: results.timing.search,
  postProcessing: results.timing.postProcess,
  candidatesEvaluated: results.debug.candidates
});
```

---

## Migration from Other Vector DBs

### From Pinecone

```javascript
// Pinecone
const pinecone = new Pinecone({ apiKey });
await pinecone.index('media').upsert([{
  id: 'inception',
  values: embedding,
  metadata: { title: 'Inception' }
}]);

// RuVector equivalent (faster + free)
await db.insert({
  id: 'inception',
  vector: embedding,
  metadata: { title: 'Inception' }
});
```

### From Weaviate

```javascript
// Weaviate GraphQL
const weaviate = await client.graphql
  .get()
  .withClassName('Media')
  .withNearVector({ vector: embedding })
  .withLimit(10)
  .do();

// RuVector Cypher (similar syntax)
const results = await db.query(`
  MATCH (media:Media)
  WITH media, vectorSimilarity(media.embedding, $query) AS score
  RETURN media
  ORDER BY score DESC
  LIMIT 10
`, { query: embedding });
```

### From Chroma

```javascript
// Chroma
const collection = await client.getCollection('media');
await collection.add({
  ids: ['inception'],
  embeddings: [embedding],
  metadatas: [{ title: 'Inception' }]
});

// RuVector (100x faster)
await db.insert({
  id: 'inception',
  vector: embedding,
  metadata: { title: 'Inception' }
});
```

---

## Troubleshooting

### Issue: Slow queries (>1ms)

**Solutions**:
1. Check dimension count: 384D recommended
2. Enable SIMD: Automatic, ensure modern CPU
3. Use batch operations instead of sequential
4. Verify HNSW index is built: `db.buildIndex()`

### Issue: Poor search results

**Solutions**:
1. Improve embeddings: Use multi-field inputs
2. Normalize vectors: `normalize: true` in embedder
3. Adjust threshold: Lower from 0.75 to 0.65
4. Add metadata filters: Combine vector + Cypher
5. Let GNN learn: Track interactions for self-improvement

### Issue: High memory usage

**Solutions**:
1. Enable compression: Automatic for cold data
2. Use smaller dimensions: 384D instead of 768D
3. Set TTL for old data: Auto-archive unused vectors
4. Use batch operations: More efficient memory allocation

---

## Best Practices

### 1. Embedding Strategy

```javascript
✅ DO: Combine multiple fields
const text = `${title} | ${plot} | ${genre} | ${mood}`;

✅ DO: Normalize vectors
const embedding = await embedder(text, { normalize: true });

❌ DON'T: Use only title
const embedding = await embedder(title);

❌ DON'T: Forget to normalize
const embedding = await embedder(text);
```

### 2. Query Optimization

```javascript
✅ DO: Use metadata filters
WHERE media.genre = "sci-fi" AND vectorSimilarity(...) > 0.75

✅ DO: Set reasonable thresholds
threshold: 0.7  // Good balance

❌ DON'T: Search entire database
// Missing WHERE clause

❌ DON'T: Use overly strict thresholds
threshold: 0.95  // Too strict, few results
```

### 3. Self-Learning

```javascript
✅ DO: Track all user interactions
await db.track({ userId, mediaId, interaction: 'view' });
await db.track({ userId, mediaId, interaction: 'like' });
await db.track({ userId, mediaId, interaction: 'skip' });

✅ DO: Let GNN optimize automatically
// No configuration needed!

❌ DON'T: Only track positive interactions
// GNN needs negative signals too
```

---

## Resources

- **Documentation**: [github.com/ruvnet/ruvector](https://github.com/ruvnet/ruvector)
- **Claude Flow Integration**: [github.com/ruvnet/claude-flow](https://github.com/ruvnet/claude-flow)
- **AgentDB Skills**: Issue [#822](https://github.com/ruvnet/claude-flow/issues/822)
- **Community**: GitHub Discussions

---

**Last Updated**: 2025-12-06
**Version**: 1.0.0
**License**: MIT
