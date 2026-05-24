# Module 06: AI System Design + Production — Progress Tracker

**Start Date:** _______________ | **Target:** 4 weeks | **Actual Completion:** _______________

---

## Week 1: Evaluation Frameworks

| Task | Done |
|------|------|
| Create a test set: 20 question + expected answer pairs about your payment policies | ☐ |
| Implement `evaluateFaithfulness()`: LLM checks if every claim is in the context | ☐ |
| Implement `evaluateAnswerRelevance()`: LLM scores 0-1 how well answer addresses question | ☐ |
| Run RAGAS-style evaluation on your Module 03 RAG system — record baseline scores | ☐ |
| Implement LLM-as-judge: judge prompt with accuracy, safety, helpfulness criteria | ☐ |
| Run judge on 20 sample answers — compare to your own manual ratings | ☐ |
| Research Q1, Q2 answered | ☐ |

---

## Week 2: Production Architecture

| Task | Done |
|------|------|
| Implement exact caching with `@Cacheable` for RAG queries | ☐ |
| Implement semantic cache: embed query → search Redis for similar cached answers | ☐ |
| Add per-user rate limiting (50 queries/hour) with Guava RateLimiter | ☐ |
| Add token counting and cost tracking: log tokens + USD cost for every LLM call | ☐ |
| Implement Resilience4j CircuitBreaker for LLM API calls | ☐ |
| Test circuit breaker: simulate API failure → verify fallback is triggered | ☐ |
| Add structured logging: every RAG query logged with full metrics | ☐ |
| Add Micrometer metrics: timer for RAG duration, counters for errors/cache hits | ☐ |
| Research Q3, Q4 answered | ☐ |

---

## Week 3: Security

| Task | Done |
|------|------|
| Implement output sanitizer: redact card numbers from AI output (`\b\d{13,19}\b`) | ☐ |
| Test indirect prompt injection: ingest a document with hidden instructions — what happens? | ☐ |
| Add document sanitization in ingestion pipeline | ☐ |
| Implement multi-tenant security: verify tenant isolation in retrieval | ☐ |
| Write security test: verify AI never returns data from another tenant's documents | ☐ |
| Document the compliance requirements for sending payment data to external APIs | ☐ |
| Research Q5, Q6 answered | ☐ |

---

## Week 4: Architecture Patterns + Design

| Task | Done |
|------|------|
| Draw the architecture diagram for your Module 03 RAG system with all production components | ☐ |
| Add observability dashboard: metrics for latency, cost, cache hit rate, error rate | ☐ |
| Write the capacity plan: how many tokens/day? what is the monthly cost? | ☐ |
| Design exercise: design a system to process 1M support tickets/day (document it) | ☐ |
| Write the AI system design document for your RAG system (all 10 sections) | ☐ |
| Answer the 10 architect graduation questions from MASTER_INDEX.md without notes | ☐ |
| Research Q7, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you design a production AI system end-to-end — including evaluation, caching, security, compliance, and observability — and explain every decision? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| RAGAS evaluation metrics (faithfulness, relevance, precision, recall) | |
| LLM-as-judge implementation | |
| Semantic caching strategy | |
| Token cost calculation and budget control | |
| Circuit breaker pattern for LLM APIs | |
| AI observability (metrics, logs, traces) | |
| Prompt injection and indirect injection defence | |
| PCI DSS / GDPR compliance for AI | |
| AI system design document | |
| RAG vs fine-tuning decision framework | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Evaluation | | |
| Week 2 — Production Architecture | | |
| Week 3 — Security | | |
| Week 4 — Design Patterns | | |
| **Total** | | |
