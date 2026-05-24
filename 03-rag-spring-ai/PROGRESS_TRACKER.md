# Module 03: RAG Systems with Spring AI — Progress Tracker

**Start Date:** _______________ | **Target:** 4 weeks | **Actual Completion:** _______________

---

## Week 1: RAG Architecture + Basic Implementation

| Task | Done |
|------|------|
| Draw the RAG pipeline on paper: ingestion phase vs query phase — label every component | ☐ |
| Implement NaiveRagService: query → vector search → build prompt → LLM → answer | ☐ |
| Configure Spring AI QuestionAnswerAdvisor and compare to manual implementation | ☐ |
| Build DocumentIngestionPipeline: support PDF and TXT files | ☐ |
| Ingest 5 payment policy documents — verify they're stored in pgvector | ☐ |
| Query the system with 10 questions — rate answer quality 1-5 for each | ☐ |
| Test the "I don't have that information" path — query for something not in docs | ☐ |
| Research Q1 answered | ☐ |

---

## Week 2: Advanced Retrieval

| Task | Done |
|------|------|
| Implement HyDE: generate hypothetical answer → embed → search | ☐ |
| Implement query rewriting: send user query to LLM for refinement before retrieval | ☐ |
| Implement multi-query retrieval: 3 query variants → merge results | ☐ |
| Add keyword search via PostgreSQL full-text search (`to_tsvector` / `plainto_tsquery`) | ☐ |
| Implement Reciprocal Rank Fusion to combine keyword + semantic results | ☐ |
| Implement LLM re-ranking: rate each retrieved chunk for relevance | ☐ |
| Compare: naive RAG vs HyDE vs hybrid search — which is most accurate on your 10 questions? | ☐ |
| Research Q2, Q3, Q4 answered | ☐ |

---

## Week 3: Conversational RAG + Failure Modes

| Task | Done |
|------|------|
| Implement ConversationalRagService with session-based history | ☐ |
| Implement query reformulation for follow-up questions | ☐ |
| Add source citations to every answer (SOURCE_1, SOURCE_2, etc.) | ☐ |
| Test: "What is the refund policy?" → "How long does THAT take?" — does it work? | ☐ |
| Trigger each RAG failure mode intentionally: retrieval failure, hallucination, context poisoning | ☐ |
| Add a faithfulness check: after each answer, verify every claim is in the retrieved context | ☐ |
| Implement multi-tenant isolation: two tenants, verify neither can see the other's docs | ☐ |
| Research Q5, Q6, Q7, Q8 answered | ☐ |

---

## Week 4: Production Application

| Task | Done |
|------|------|
| Build PolicyQAController: /ingest (admin) + /ask + /chat endpoints | ☐ |
| Add Spring Security: admin-only for ingestion, auth required for queries | ☐ |
| Add observability: log retrieval latency, chunks retrieved, LLM latency per call | ☐ |
| Write integration test: ingest document → query → verify answer references correct source | ☐ |
| Measure baseline with 20 test questions + expected answers (accuracy score) | ☐ |
| Try tuning: reduce chunk size to 256 — does accuracy improve? | ☐ |
| Write a 1-page "RAG system design" document from memory (architecture diagram + key decisions) | ☐ |
| Research Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build a complete RAG system from scratch (ingestion → retrieval → generation → citations) that answers questions about your own documents? Can you diagnose retrieval failures? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| RAG pipeline architecture (ingestion + query phases) | |
| Spring AI QuestionAnswerAdvisor configuration | |
| Document ingestion pipeline (PDF → chunks → embed → store) | |
| HyDE query transformation | |
| Hybrid search (keyword + semantic + RRF) | |
| Re-ranking retrieved results | |
| Conversational RAG with query reformulation | |
| Source citation and transparency | |
| RAG failure diagnosis (retrieval failure, hallucination, poisoning) | |
| Multi-tenant RAG with security isolation | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Basic RAG | | |
| Week 2 — Advanced Retrieval | | |
| Week 3 — Conversational + Failures | | |
| Week 4 — Production App | | |
| **Total** | | |
