# Module 02: Embeddings + Vector Databases
**Duration:** 3 weeks (2 hours/day)
**Goal:** Understand embeddings deeply and build semantic search systems using pgvector and Qdrant.

---

## The Big Idea

```
Traditional search: find documents containing the word "payment declined"
Semantic search:    find documents with similar MEANING to "payment declined"
                    → finds: "transaction rejected", "card not accepted",
                              "authorization failed", "purchase unsuccessful"
                    even if those exact words don't appear in the query

How? Convert text to a vector (list of ~1536 numbers).
Similar text → vectors that are mathematically close to each other.
Search = find vectors close to the query vector.

This is the foundation of RAG, document search, recommendation systems,
and any AI feature that needs to understand language semantically.
```

---

## Week 1: Understanding Embeddings

**Monday — What Is an Embedding?**
```java
// An embedding is a list of floating-point numbers that represents text meaning.
// text-embedding-3-small: 1536 numbers per piece of text
// text-embedding-3-large: 3072 numbers per piece of text
// nomic-embed-text (local): 768 numbers per piece of text

// Example:
// "Payment declined" → [0.023, -0.412, 0.891, 0.034, -0.227, ... (1536 total)]
// "Transaction rejected" → [0.031, -0.398, 0.874, 0.041, -0.219, ... (1536 total)]
// "The weather is nice" → [-0.812, 0.234, -0.109, 0.677, 0.923, ... (1536 total)]

// "Payment declined" and "Transaction rejected" have SIMILAR vectors (close in space)
// "The weather is nice" has a VERY DIFFERENT vector (far away in space)

// Similarity measured by cosine similarity:
// Score: -1 (opposite meaning) to 1 (same meaning)
// "Payment declined" vs "Transaction rejected": ~0.92 (very similar)
// "Payment declined" vs "The weather is nice": ~0.12 (unrelated)

// Generate embeddings with Spring AI:
@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public float[] embed(String text) {
        EmbeddingResponse response = embeddingModel.embedForResponse(List.of(text));
        return response.getResults().get(0).getOutput();
    }

    public List<float[]> embedBatch(List<String> texts) {
        EmbeddingResponse response = embeddingModel.embedForResponse(texts);
        return response.getResults().stream()
            .map(EmbeddingResultMetadata::getOutput)
            .collect(Collectors.toList());
    }
}
```

**Tuesday — Cosine Similarity and Distance Metrics**
```java
// Cosine similarity: the standard metric for text embeddings
// Measures the ANGLE between two vectors (ignores magnitude)
// 1.0 = identical direction (same meaning)
// 0.0 = perpendicular (unrelated)
// -1.0 = opposite direction (opposite meaning)

public static double cosineSimilarity(float[] a, float[] b) {
    double dotProduct = 0;
    double normA = 0;
    double normB = 0;
    for (int i = 0; i < a.length; i++) {
        dotProduct += a[i] * b[i];
        normA += a[i] * a[i];
        normB += b[i] * b[i];
    }
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}

// Other distance metrics:
// Euclidean distance (L2): actual distance between vectors in space
//   Good for: absolute position matters
//   Bad for: text (magnitude varies with text length)
// Dot product: cosine similarity without normalising
//   Use when: embeddings are already normalised (many models do this)
// Manhattan distance (L1): sum of absolute differences
//   Less common for text embeddings

// Rule for choosing: for text embeddings → use COSINE SIMILARITY (always)
// pgvector operators:
// <->  L2 distance      (less common for text)
// <#>  Negative dot product (common for normalised vectors)
// <=>  Cosine distance  (use this for text: 1 - cosine_similarity)
```

**Wednesday — Chunking Strategies**
```java
// Documents must be split into chunks before embedding.
// The chunk is what gets embedded and stored in the vector database.

// Chunking strategies:

// 1. Fixed-size chunking (simplest):
//    Split every N tokens, with M token overlap
//    Overlap prevents losing context at chunk boundaries

// 2. Sentence-based chunking (better quality):
//    Split at sentence boundaries
//    Preserve complete thoughts

// 3. Semantic chunking (best quality, most complex):
//    Use another LLM to identify natural topic breaks
//    Expensive but highest quality

// 4. Recursive chunking (Spring AI default):
//    Split on: paragraphs → sentences → words → characters
//    Falls back to smaller delimiter when chunk too large

// Spring AI TokenTextSplitter:
TokenTextSplitter splitter = new TokenTextSplitter(
    512,   // target chunk size (tokens)
    100,   // minimum chunk size
    5,     // max number of sentences per chunk
    10000, // max number of characters
    true   // keep separator
);

List<Document> documents = List.of(
    new Document("Payment processing guide...\n\nSection 1: Introduction...\n...")
);

List<Document> chunks = splitter.apply(documents);
// Each chunk: smaller piece of the original document, with overlap

// Chunking rules of thumb:
// Too small (< 100 tokens): not enough context, bad retrieval quality
// Too large (> 1000 tokens): too much noise, costs more, lower precision
// Sweet spot: 200-500 tokens for most use cases
// Overlap: 10-20% of chunk size (prevents context loss at boundaries)

// METADATA is critical: each chunk must carry metadata from the source document
Document doc = new Document(
    chunkText,
    Map.of(
        "source", "payment-policy-v3.pdf",
        "page", 5,
        "section", "Fraud Prevention",
        "created_at", LocalDate.now().toString(),
        "document_type", "POLICY"
    )
);
```

**Thursday — Spring AI pgvector Setup**
```sql
-- Enable pgvector extension in PostgreSQL:
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table to store embeddings:
CREATE TABLE document_embeddings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content     TEXT NOT NULL,               -- the chunk text
    embedding   vector(1536) NOT NULL,       -- the embedding (1536 for text-embedding-3-small)
    metadata    JSONB,                       -- source, page, section, etc.
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast nearest-neighbor search:
-- IVFFlat: approximate, fast, good for most use cases
CREATE INDEX ON document_embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- lists = sqrt(row_count) is a good rule of thumb

-- HNSW: higher memory, faster queries, better recall (pgvector 0.5+)
CREATE INDEX ON document_embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Similarity search query:
SELECT
    id,
    content,
    metadata,
    1 - (embedding <=> '[0.023, -0.412, ...]'::vector) AS similarity
FROM document_embeddings
ORDER BY embedding <=> '[0.023, -0.412, ...]'::vector
LIMIT 5;

-- Filtered similarity search (only search within a category):
SELECT id, content, metadata,
       1 - (embedding <=> :queryEmbedding::vector) AS similarity
FROM document_embeddings
WHERE metadata->>'document_type' = 'POLICY'
  AND metadata->>'section' = 'Fraud Prevention'
ORDER BY embedding <=> :queryEmbedding::vector
LIMIT 10;
```

```yaml
# application.yml for Spring AI + pgvector
spring:
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
    openai:
      embedding:
        options:
          model: text-embedding-3-small
```

**Friday — Ingestion Pipeline**
```java
// Full document ingestion pipeline: PDF → chunks → embeddings → pgvector

@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;              // Spring AI abstraction
    private final TokenTextSplitter splitter;
    private final DocumentReader pdfReader;

    @Transactional
    public void ingestPdf(MultipartFile file, Map<String, Object> metadata) throws IOException {
        // 1. Read PDF
        Resource pdfResource = new ByteArrayResource(file.getBytes());
        List<Document> rawDocuments = new PagePdfDocumentReader(pdfResource)
            .get();

        // 2. Add metadata to each document
        rawDocuments = rawDocuments.stream()
            .map(doc -> {
                Map<String, Object> enrichedMetadata = new HashMap<>(doc.getMetadata());
                enrichedMetadata.putAll(metadata);
                enrichedMetadata.put("ingested_at", LocalDateTime.now().toString());
                enrichedMetadata.put("filename", file.getOriginalFilename());
                return new Document(doc.getContent(), enrichedMetadata);
            })
            .collect(Collectors.toList());

        // 3. Split into chunks
        List<Document> chunks = splitter.apply(rawDocuments);
        log.info("Split {} pages into {} chunks", rawDocuments.size(), chunks.size());

        // 4. Embed and store (Spring AI handles embedding + storage)
        vectorStore.add(chunks);
        // Spring AI:
        // a. Calls embedding API for each chunk (batched)
        // b. Stores content + embedding + metadata in pgvector table

        log.info("Ingested {} chunks into vector store", chunks.size());
    }

    // Similarity search:
    public List<Document> search(String query, int topK, Map<String, Object> filters) {
        SearchRequest searchRequest = SearchRequest.query(query)
            .withTopK(topK)
            .withSimilarityThreshold(0.7)  // only return results > 70% similar
            .withFilterExpression(buildFilter(filters));

        return vectorStore.similaritySearch(searchRequest);
    }

    private FilterExpressionTextParser.Filter buildFilter(Map<String, Object> filters) {
        // Example: only search in POLICY documents from 2024
        // document_type == 'POLICY' AND year >= 2024
        return FilterExpressionTextParser.fromText(
            "document_type == 'POLICY'"
        );
    }
}
```

---

## Week 2: Qdrant Vector Database

**Monday–Wednesday — Qdrant Setup and Operations**
```yaml
# Docker Compose for Qdrant:
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"   # REST API
      - "6334:6334"   # gRPC
    volumes:
      - qdrant_data:/qdrant/storage

# application.yml:
spring:
  ai:
    vectorstore:
      qdrant:
        host: localhost
        port: 6334
        collection-name: payment-docs
        use-tls: false
        api-key: ${QDRANT_API_KEY:}
```

```java
// Qdrant collection creation (via REST):
// POST http://localhost:6333/collections/payment-docs
{
    "vectors": {
        "size": 1536,
        "distance": "Cosine"
    }
}

// Qdrant advantages over pgvector:
// ✅ Built for vector search (pgvector is an extension)
// ✅ Filtering is extremely fast (uses payload indexes)
// ✅ HNSW always on (no configuration needed)
// ✅ Quantization (compress vectors to save memory)
// ✅ REST + gRPC API
// ✅ Docker-ready, Kubernetes-native

// Use Qdrant when:
// - New project with no existing PostgreSQL dependency
// - High query volume (>1000 searches/second)
// - Large collections (>10M vectors)
// - Need filtering on many metadata fields

// Use pgvector when:
// - You already use PostgreSQL for your data
// - You want ACID transactions across your data and embeddings
// - Smaller scale (<5M vectors)
// - Don't want another service to manage
```

**Thursday–Friday — Embedding Models Comparison**
```
Model                    Dimensions  Speed    Quality   Cost    Private
──────────────────────────────────────────────────────────────────────
text-embedding-3-small   1536        Fast     Good      Low     No (OpenAI)
text-embedding-3-large   3072        Medium   Best      Medium  No (OpenAI)
nomic-embed-text         768         Fast     Good      Free    Yes (Ollama)
mxbai-embed-large        1024        Medium   Very Good Free    Yes (Ollama)
all-minilm-l6-v2         384         Fastest  OK        Free    Yes (local)

For fintech (data privacy):
→ nomic-embed-text via Ollama (runs locally, data never leaves your infra)
→ Azure OpenAI embeddings (data stays in your Azure tenant)

For best quality at scale:
→ text-embedding-3-large (best accuracy, higher cost)

Sweet spot for most Java apps:
→ text-embedding-3-small (good quality, low cost, fast)
```

---

## Week 3: Semantic Search Application

**Build a Payment Policy Search Engine**
```java
// Full application: search through 100 payment policy documents
// Query: "what happens when a payment is declined due to fraud?"
// Returns: relevant policy sections, ranked by semantic similarity

@RestController
@RequestMapping("/api/search")
public class PolicySearchController {

    @PostMapping
    public SearchResponse search(@RequestBody SearchRequest request) {
        // 1. Validate query
        if (request.query().length() > 1000) {
            throw new IllegalArgumentException("Query too long");
        }

        // 2. Semantic search
        List<Document> results = vectorStore.similaritySearch(
            SearchRequest.query(request.query())
                .withTopK(5)
                .withSimilarityThreshold(0.65)
        );

        // 3. Format results
        List<SearchResult> formatted = results.stream()
            .map(doc -> new SearchResult(
                doc.getContent(),
                (String) doc.getMetadata().get("source"),
                (String) doc.getMetadata().get("section"),
                (Double) doc.getMetadata().get("score")
            ))
            .collect(Collectors.toList());

        return new SearchResponse(formatted, results.size());
    }

    record SearchResult(String content, String source, String section, Double score) {}
    record SearchResponse(List<SearchResult> results, int totalFound) {}
}
```
