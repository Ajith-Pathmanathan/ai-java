# Module 02: Embeddings + Vector Databases — Progress Tracker

**Start Date:** _______________ | **Target:** 3 weeks | **Actual Completion:** _______________

---

## Week 1: Understanding Embeddings

| Task | Done |
|------|------|
| Generate your first embedding: call `embeddingModel.embedForResponse()` for 3 sentences — print the float arrays | ☐ |
| Implement `cosineSimilarity(float[] a, float[] b)` manually in Java — verify "payment declined" vs "transaction rejected" > 0.85 | ☐ |
| Compare similarity scores: same category vs different category sentences — see the numbers in action | ☐ |
| Enable pgvector: `CREATE EXTENSION IF NOT EXISTS vector` — confirm it works | ☐ |
| Create `document_embeddings` table with `vector(1536)` column and HNSW index | ☐ |
| Write TokenTextSplitter with 512 token chunks, 100 token overlap — test on a long string | ☐ |
| Implement DocumentIngestionService: read text → split → embed → store in pgvector | ☐ |
| Research Q1, Q2 answered | ☐ |

---

## Week 2: Qdrant Vector Database

| Task | Done |
|------|------|
| Run Qdrant via Docker Compose — confirm REST API at localhost:6333 | ☐ |
| Create a Qdrant collection `payment-docs` with size=1536, distance=Cosine | ☐ |
| Configure Spring AI to use Qdrant (application.yml) | ☐ |
| Ingest 20 sample documents into Qdrant via `vectorStore.add()` | ☐ |
| Run a similarity search — print top 5 results with scores | ☐ |
| Add metadata filtering: only search within documents of type "POLICY" | ☐ |
| Compare results: pgvector vs Qdrant for the same query — same results? | ☐ |
| Research Q3, Q4, Q5 answered | ☐ |

---

## Week 3: Semantic Search Application

| Task | Done |
|------|------|
| Ingest 10 PDF documents (use sample payment policy PDFs from the internet) | ☐ |
| Build PolicySearchController with POST /api/search endpoint | ☐ |
| Add similarity threshold filtering (0.65) — test what gets filtered out | ☐ |
| Add metadata to search results (source document, section) | ☐ |
| Test with 10 different queries — document recall and relevance | ☐ |
| Tune chunk size: test 256, 512, 1024 tokens — which gives best results? | ☐ |
| Add tenant isolation: metadata filter for tenant_id in every search | ☐ |
| Research Q6, Q7, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build a semantic search system that ingests PDF documents, stores them in a vector database, and returns relevant results with metadata? Can you explain when to use pgvector vs Qdrant? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| Generating embeddings with Spring AI | |
| Cosine similarity (formula + implementation) | |
| Chunking strategy selection | |
| pgvector setup (extension, table, HNSW index) | |
| Qdrant setup and collection management | |
| Document ingestion pipeline (PDF → chunks → embed → store) | |
| Filtered similarity search | |
| Embedding model selection for fintech | |
| Tuning chunk size for retrieval quality | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Embeddings + pgvector | | |
| Week 2 — Qdrant | | |
| Week 3 — Semantic Search App | | |
| **Total** | | |
