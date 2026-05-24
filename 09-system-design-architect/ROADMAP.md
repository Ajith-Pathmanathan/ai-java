# Module 09: System Design for Architect Level
**Duration:** 4 weeks (2 hours/day)
**Goal:** Master the system design skills needed for senior architect interviews and real-world design decisions.

---

## The Big Idea

```
System design is what separates an architect from a developer.
Anyone can code a feature. Architects can design systems that:
  - Handle millions of users
  - Survive failures
  - Scale economically
  - Maintain correctness under load

In interviews: "Design Twitter / Uber / a payment system"
In real life:  "We're launching in 3 new markets, will the system hold?"

The same mental model works for both.
```

---

## Week 1: System Design Framework

**Monday — The Framework (Use This Every Time)**
```
Every system design uses this 6-step framework:

1. REQUIREMENTS CLARIFICATION (5 minutes)
   - Functional: what must the system DO?
   - Non-functional: scale, latency, availability, consistency

2. CAPACITY ESTIMATION (5 minutes)
   - Daily active users
   - Requests per second (peak)
   - Storage growth per day
   - Bandwidth requirements

3. HIGH-LEVEL DESIGN (10 minutes)
   - Core components: client → LB → services → storage
   - Data flow: how data moves through the system

4. DETAILED DESIGN (15 minutes)
   - Database schema
   - APIs
   - Key algorithms
   - Trade-offs made

5. BOTTLENECKS + TRADE-OFFS (5 minutes)
   - Where does it fail at scale?
   - What are the consistency trade-offs?
   - What would you change with 10x the scale?

6. AI INTEGRATION (bonus in 2025)
   - Where does AI add value in this system?
   - RAG, agents, embeddings — which fits?
```

**Tuesday–Wednesday — Capacity Estimation**
```
Master these numbers (memorize them):

LATENCY:
  Memory access:          0.1ms
  SSD random read:        0.1ms
  PostgreSQL index query: 1ms
  Vector similarity (ANN): 5-50ms
  LLM inference (GPT-4o): 1000-5000ms
  Network (same DC):      0.5ms
  Network (cross DC):     50-150ms

THROUGHPUT:
  Single PostgreSQL:        10,000 reads/s, 1,000 writes/s
  Redis:                    100,000 ops/s
  Single Kafka broker:      100,000 msg/s
  Single Spring Boot pod:   500-2,000 req/s (no AI)
  Single Spring Boot + LLM: 10-50 req/s (LLM latency dominates)

STORAGE:
  1 embedding (1536 float32): 6KB
  1M embeddings:              6GB
  pgvector index (HNSW):      1.5-3x the vector data
  1 payment record:           ~1KB
  1M payment records/day:     ~1GB/day

EXAMPLE: Design for 1 million RAG queries/day
  Queries/day:    1,000,000
  Queries/second: ~12 average, ~120 peak (10x factor)
  Cache hit (40%): 48/second reach LLM
  LLM tokens/query: 3,000 (input) + 500 (output)
  Total tokens/day: 1M × 3,500 × 0.6 = 2.1B tokens
  Cost (GPT-4o):   $10.50/day = $315/month
  With caching:    ~$190/month
```

**Thursday–Friday — Design #1: Payment AI Assistant**
```
DESIGN: Payment AI Assistant that answers merchant questions using their transaction data

Requirements:
  - 100K merchants, each with up to 10 years of transaction history
  - Questions about transactions, trends, anomalies
  - P95 latency < 3 seconds
  - 99.9% availability
  - Data isolation: merchant A cannot access merchant B's data

Architecture:

  Merchant App → API Gateway → Auth (JWT, per-merchant)
                                     ↓
                              Payment AI Service
                              ├─ RAG Query Engine
                              │   ├─ Redis (semantic cache, keyed by merchant_id + query)
                              │   ├─ pgvector (per-merchant embeddings, filtered by tenant_id)
                              │   └─ Azure OpenAI GPT-4o
                              └─ Analytics Engine
                                  ├─ Pre-computed aggregations (daily Spark job)
                                  └─ PostgreSQL (transaction data)

Key decisions:
  1. Tenant isolation: every pgvector query filters by merchant_id
  2. Semantic cache: keyed by (merchant_id, query_embedding) — prevents cross-tenant hits
  3. Azure OpenAI: data stays in EU (GDPR compliance)
  4. Pre-computed aggregations: merchants often ask "what are my top 10 merchants by volume?"
     → pre-compute daily, not on every query
  5. Async enrichment: when merchant uploads transaction export → Kafka event
     → background ingestion → pgvector populated over 1-5 minutes
```

---

## Week 2: Classic System Design Problems

**Monday — Design a URL Shortener (Warm-up)**
```
REQUIREMENTS: 100M URLs shortened/day, 10B reads/day, 5-year data retention

CAPACITY:
  Writes: 100M/day = 1,160/second = 2,320/sec peak (2x)
  Reads:  10B/day = 115,700/second = 231,400/sec peak

  Storage: 100M URLs/day × 365 × 5 years = 182.5B URLs
           Each URL: key(7 chars) + original URL(200 bytes) + metadata = ~300 bytes
           Total: 55TB over 5 years

  Reads are 10,000x more frequent than writes → heavy cache on read path

DESIGN:
  Write path:  API Server → Generate short key (Base62, 7 chars = 62^7 = 3.5T unique)
               → Store in Redis (TTL = 24h, hot URLs)
               → Async write to PostgreSQL (persistent)

  Read path:   DNS → CDN (edge cache, cache short URLs) → Redis → PostgreSQL
               95% of reads served from CDN/Redis

  Key generation: MD5(URL) → take first 7 chars → collision check → retry
                  Or: distributed counter (Snowflake ID → Base62)

TRADE-OFFS:
  - PostgreSQL handles 10K reads/sec + Redis handles 100K/sec → together handle peak
  - CDN reduces latency to <10ms for cached URLs
  - Analytics (clicks per URL) → Kafka → async aggregation (Spark or Flink)
  - Expiry: TTL in Redis + soft delete in PostgreSQL (GDPR compliance)
```

**Tuesday — Design Twitter/X Feed**
```
REQUIREMENTS: 300M DAU, 500M tweets/day, read tweet feed < 300ms

CAPACITY:
  Writes: 500M/day = 5,800/second peak ~12,000/second
  Reads: 300M users × 50 feed reads/day = 15B reads/day = 173,600/second

DESIGN:
  Write (tweet creation):
    User → API Server → Kafka(tweet-created) → Fanout Service
    Fanout: for each follower → push tweet ID to their feed list (Redis List)
    Problem: celebrities have 100M followers → fanout takes too long

    Solution (hybrid fanout):
    - Regular users (< 1M followers): push fanout (write to all follower feeds)
    - Celebrities (> 1M followers): pull on read (don't fanout, add to timeline at read)

  Read (get feed):
    User → API Server → Feed Service → Redis (pre-built feed list, top 1000 tweet IDs)
           → Fetch tweet details (cache in Redis)
           → Merge celebrity tweets (pull from celebrity tweets list) → Sort by time
           → Return to user

  Storage:
    Tweets: Cassandra (high write throughput, time-series, AP system — eventual consistency ok for tweets)
    User graph (followers): Graph DB or PostgreSQL
    Feed cache: Redis (sorted set, tweet IDs ordered by timestamp)
    Media: S3 + CDN (never store images in the database)

AI addition:
  - Tweet ranking: instead of chronological feed, use ML to rank by engagement probability
  - Content moderation: Kafka consumer → LLM classifies each tweet for policy violations
  - Recommendation: "Tweets you might like" → embedding similarity of user interest profile vs tweet embeddings
```

**Wednesday — Design a Ride-Sharing Platform (Uber)**
```
REQUIREMENTS: 10M rides/day, match driver to rider < 5 seconds, real-time location tracking

CAPACITY:
  10M rides/day = 116 rides/second average
  Peak: 10x = 1,160 rides/second
  Driver locations: 5M active drivers × update every 5 seconds = 1M location updates/second

DESIGN:
  Location tracking:
    Driver app → WebSocket → Location Service → Redis Geo (GEOADD lat/lng)
    Redis GEO: store driver locations, query nearby drivers (GEORADIUS)
    Partition by city to scale (NYC drivers in NYC Redis cluster)

  Matching (< 5 seconds):
    Rider requests → Matching Service → Redis GEO nearest drivers (within 5km)
                  → Rank by: distance, rating, ETA, car type
                  → Send request to best driver (push via WebSocket)
                  → Driver accepts/declines (timeout 15 seconds)
                  → If decline: try next driver

  Ride lifecycle events → Kafka → multiple consumers:
    - Billing service (charge at ride end)
    - Analytics service (surge pricing)
    - Safety service (background checks)

  Surge pricing:
    Supply (available drivers) + Demand (ride requests) → calculated every 30 seconds
    High demand → AI recommends surge multiplier based on historical patterns

AI addition:
  - ETA prediction: ML model (trained on historical trip data + current traffic) → more accurate than Google Maps alone
  - Surge pricing: ML model (demand patterns, events, weather) → optimize price
  - Safety: route deviation detection (driver takes wrong route) → alert
  - Driver matching: LLM-powered "for accessible vehicle requests, match with drivers who have wheelchair-equipped cars"
```

**Thursday–Friday — Design #2: AI Fraud Detection**
```
DESIGN: Real-time fraud detection for payment transactions
        Decision required in < 500ms (payment authorization timeout)

Requirements:
  1M transactions/day, detect fraud in < 500ms, 99.9% uptime
  False positive rate < 0.1% (blocking legitimate transactions is bad)

DESIGN:

  Transaction → API Gateway → Fraud Check Service → Decision (APPROVE/DECLINE/REVIEW)

  Fraud Check Service (must complete in < 200ms — leave 300ms buffer):
    Step 1: Rule-based checks (< 5ms) — PostgreSQL, indexed by card ID
      - Is this card on the blocklist?
      - Amount > 3x card's average transaction?
      - Multiple transactions in 1 minute?

    Step 2: ML model scoring (< 50ms) — pre-trained model, in-memory inference
      - XGBoost or LightGBM model (fast inference)
      - Features: amount, merchant category, time of day, country, device fingerprint
      - Score 0-100 (fraud probability)

    Step 3: AI (LLM) analysis (< 100ms) — only for borderline cases (score 40-60)
      - Agent uses tools to check: customer history, merchant reputation, recent patterns
      - LLM decision: APPROVE / REVIEW / DECLINE + reason

  Decision logic:
    Score 0-40: APPROVE (rule-based pass → ML low risk)
    Score 40-60: LLM analysis (borderline)
    Score 60-100: DECLINE or flag for human review

  Storage:
    Transaction history: TimescaleDB (time-series, partitioned by date)
    Rule blocklist: Redis (in-memory for < 1ms lookup)
    ML model: loaded in-memory (no DB call needed)
    Fraud decisions: PostgreSQL (audit trail for regulatory compliance)

  Why NOT use LLM for every transaction:
    LLM = 1-3 seconds = too slow for 500ms requirement
    LLM used ONLY for borderline cases (20% of transactions)
    Keeps P95 latency within budget
```

---

## Week 3: AI-Specific System Designs

**Monday–Tuesday — Design a RAG System at Scale**
```
DESIGN: Document search system for 10M legal documents, 50K users

(Already covered in Module 03 — apply the framework formally here)

Requirements:
  - Ingest 100K new documents/day
  - Search latency < 2 seconds (including LLM generation)
  - Maintain 5 years of documents
  - Per-user access control (lawyers can only see their firm's documents)

Capacity:
  Storage: 5M docs × 5 years = 50M docs × 6KB (embeddings) = 300GB embeddings
  Index: HNSW 1.5x = 450GB total vector storage → Qdrant cluster (3 nodes)
  Ingestion: 100K/day = 1.2 docs/second average
  Queries: 50K users × 20 searches/day = 1M queries/day = 12/second average, 120/second peak

Architecture:
  Ingestion pipeline (async, Kafka):
    Document upload → S3 → Kafka → Ingestion Workers (Spring Boot, 10 pods)
    → Tika (text extraction) → tokenizer → embeddings (batch call to OpenAI)
    → Qdrant (upsert with firm_id metadata)

  Query pipeline (synchronous, < 2 seconds):
    User query → Auth → extract firm_id from JWT
    → Redis semantic cache check (keyed by firm_id + query_embedding, TTL 1h)
    → Qdrant (filter by firm_id) → top-5 chunks
    → GPT-4o (grounded generation, max 500 tokens output)
    → Response with source citations

Key decisions:
  Access control: every Qdrant query filters by firm_id → enforced at search time
  Qdrant cluster: 3 nodes, 1 replica factor, HNSW index
  OpenAI embeddings: batch at ingestion, single call at query
  Semantic cache: same firm + same question → cache hit (1hr TTL)
  GPT-4o max_tokens=500: limits generation cost, forces concise answers
```

**Wednesday — Design an AI Agent Platform**
```
DESIGN: A platform where businesses can build custom AI agents with tools

(Think: an enterprise version of Claude Agents or GPT Actions)

Key components:
  - Tool registry: businesses register their tools (REST API endpoints)
  - Agent builder: non-technical staff define agent behavior via UI
  - Execution engine: runs agent workflows (ReAct loop)
  - Monitoring: track every agent run

Architecture:
  Tool Registry → PostgreSQL (tool definitions: name, endpoint, auth, schema)
  Agent Config  → PostgreSQL (system prompt, allowed tools, model, temperature)
  
  Execution:
    User query → Agent Runner Service
    → Load agent config → build ChatClient with tools
    → ReAct loop: reason → call tool → observe → repeat
    → Stream response to user
  
  Observability:
    Every agent run → Kafka (agent-runs topic) → ClickHouse (analytics)
    Metrics: tool call count, latency, success rate, cost per run
    
  Multi-tenancy:
    Each business has a tenant_id. Agent configs, tools, and runs are isolated.
    Tools registered per tenant. Agent can only call tools in its tenant's tool registry.
```

**Thursday–Friday — Design Practice and Review**
```
PRACTICE: Design these systems in 45 minutes each (use the 6-step framework)

1. Design a notification system (email/SMS/push) for 500M users
   Hint: fan-out (like Twitter), Kafka, per-channel workers

2. Design a distributed cache (like Redis)
   Hint: consistent hashing, LRU eviction, replication

3. Design a video streaming service (like YouTube)
   Hint: CDN, chunked video, adaptive bitrate, metadata store

4. Design a global payments platform (like Stripe)
   Hint: idempotency keys, dual-write, currency conversion, regulatory isolation

5. Design a search engine for code (like GitHub code search)
   Hint: inverted index, Elasticsearch, embedding-based semantic search layer

FRAMEWORK CHECKLIST FOR EVERY DESIGN:
  ☐ Functional requirements clarified (3-4 features)
  ☐ Non-functional requirements stated (scale, latency, availability)
  ☐ Capacity estimated (RPS, storage, cost)
  ☐ High-level diagram drawn
  ☐ Database choice justified (SQL vs NoSQL + why)
  ☐ At least one scaling bottleneck identified
  ☐ At least one trade-off discussed
  ☐ AI integration considered (where does AI add value?)
```

---

## Week 4: Architect Communication

**Monday–Wednesday — Explaining Technical Decisions**
```
Architects don't just design systems — they communicate decisions.

Rule 1: Speak the listener's language
  To engineers:      "We use HNSW because it gives O(log N) query time vs O(N) brute force"
  To tech leads:     "HNSW gives us 50ms retrieval vs 5 seconds brute force at 1M vectors"
  To product manager: "This approach means search is fast enough — users don't wait"
  To CTO:            "This choice lets us grow to 10M documents without re-architecting"

Rule 2: Lead with the decision, then the reason
  WRONG: "We could use IVFFlat or HNSW, both have different trade-offs in terms of..."
  RIGHT: "We chose HNSW. It's faster at query time and has better recall. The trade-off
          is higher memory, but that's fine given our 10M document target."

Rule 3: Know the counter-argument
  "Some teams use pgvector for this. We chose Qdrant because we need 50K QPS — pgvector
   would need extensive tuning to get there. If we were < 10K QPS, pgvector would have
   been simpler since we already run PostgreSQL."

Rule 4: Use numbers, not adjectives
  WRONG: "Vector search is fast"
  RIGHT: "Vector search with HNSW returns results in 10-50ms at 1M vectors"
```

**Thursday–Friday — Architecture Decision Records**
```
Architecture Decision Record (ADR): a short document recording architectural decisions
Used by top engineering teams to avoid re-litigating past decisions

ADR Template:
  Title:   ADR-042: Use Qdrant instead of pgvector for document embeddings
  Status:  Accepted / Superseded / Deprecated
  Context: We need vector search for 10M documents at 50K QPS.
           Our current PostgreSQL setup cannot handle this throughput.
  Decision: We will use Qdrant in a 3-node cluster.
  Consequences:
    Positive: 50K QPS with < 50ms p99 latency. Payload filtering is 3x faster than pgvector.
    Negative: A new service to manage. Team needs to learn Qdrant.
    Risks: Qdrant is less mature than PostgreSQL. Mitigated by strong community + Qdrant Cloud.
  Alternatives considered:
    pgvector: Rejected — tested at 50K QPS, p99 latency was 800ms.
    Weaviate:  Rejected — more complex to operate, similar performance to Qdrant.

Write ADRs for EVERY significant technical decision in your team.
Future engineers will thank you.
When someone asks "why do we use X?" — the ADR has the answer.
```
