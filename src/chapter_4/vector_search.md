# Vector Search

Embedding models turn text, images, or users into **vectors** (arrays of floats). "Similar" items have vectors that are close together. Redis can store these vectors and answer **k-nearest-neighbor (KNN)** queries in memory, which makes it a popular low-latency vector store for RAG, semantic caching, and recommendations.

There are two ways to do it in Redis today:

1. **Vector fields in the Query Engine** (`FT.CREATE ... VECTOR`) — mature, supports hybrid queries (vector + filters). Available in Redis Stack and Redis 8.
2. **Vector Sets** (`VADD` / `VSIM`) — a newer native data type introduced in Redis 8 (initially as beta), modeled on Sorted Sets but with vectors instead of scores.

## Vector fields with FT.CREATE

```bash
# Hash documents with a 384-dim FLOAT32 embedding, HNSW index, cosine distance
127.0.0.1:6379> FT.CREATE idx:docs ON HASH PREFIX 1 doc: SCHEMA title TEXT category TAG embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 384 DISTANCE_METRIC COSINE
OK

# Write a document (embedding must be raw little-endian FLOAT32 bytes, usually from a client library)
HSET doc:1 title "Cache stampede prevention" category "redis" embedding <1536 bytes>

# Top 5 nearest docs in category "redis" to a query vector
127.0.0.1:6379> FT.SEARCH idx:docs "(@category:{redis})=>[KNN 5 @embedding $vec AS score]" PARAMS 2 vec <query bytes> SORTBY score RETURN 2 title score DIALECT 2
```

- **HNSW** is an approximate index: fast, slightly less accurate. **FLAT** is brute-force: exact, fine for small datasets (tens of thousands of vectors).
- `DIALECT 2` is required for the `PARAMS` / KNN syntax.

## Vector Sets (Redis 8)

```bash
# VADD key VALUES <dim> <floats...> element
127.0.0.1:6379> VADD movies VALUES 3 0.1 0.9 0.2 "inception"
(integer) 1
127.0.0.1:6379> VADD movies VALUES 3 0.15 0.85 0.25 "interstellar"
(integer) 1
127.0.0.1:6379> VADD movies VALUES 3 0.9 0.1 0.1 "notting-hill"
(integer) 1

# Most similar elements to an existing element
127.0.0.1:6379> VSIM movies ELE "inception" WITHSCORES COUNT 2
1) "inception"
2) "1"
3) "interstellar"
4) "0.99..."
```

Vector Sets are simpler for "one collection of vectors" use cases; the Query Engine is better when you need filters, full-text, and many fields together. Check your server version—Vector Sets don't exist before Redis 8.

## Practical patterns

### 1) RAG retrieval

Chunk documents, embed them, store them as Hash/JSON with a vector field, and `KNN` search at query time. Tag/numeric filters (tenant, language, date) in the same query avoid post-filtering.

### 2) Semantic cache for LLM calls

Embed each prompt; before calling the LLM, search for a previous prompt within a distance threshold and return its cached answer. Combine with a TTL so answers don't go stale.

### 3) Recommendations

"Users who liked this also liked…" as a nearest-neighbor query over item embeddings.

## Pitfalls

- **Memory.** 1M vectors × 768 dims × 4 bytes ≈ 3GB *before* HNSW graph overhead. Lower dimensions or quantization (where supported) matter a lot.
- **Dimension and metric must match the model.** Mixing embeddings from two models in one index silently produces garbage.
- **Approximate means approximate.** HNSW recall depends on parameters (`M`, `EF_CONSTRUCTION`, `EF_RUNTIME`); measure recall on your data.
- **Semantic cache thresholds** that are too loose return wrong answers confidently. Start strict.

## Things to think about

1. Redis is a strong fit when vectors need millisecond latency next to data you already keep in Redis.
2. Use the Query Engine for hybrid search, Vector Sets for simple similarity collections on Redis 8.
3. Budget RAM for vectors explicitly—they dominate memory quickly.
