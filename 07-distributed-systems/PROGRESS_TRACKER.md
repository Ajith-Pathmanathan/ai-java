# Module 07: Distributed Systems + Event-Driven — Progress Tracker

**Start Date:** _______________ | **Target:** 4 weeks | **Actual Completion:** _______________

---

## Week 1: Apache Kafka

| Task | Done |
|------|------|
| Run Kafka locally via Docker Compose (Kafka + Zookeeper) | ☐ |
| Add spring-kafka dependency — configure producer in application.yml | ☐ |
| Create a `PaymentEventProducer` — publish `PaymentEvent` to `payment-events` topic | ☐ |
| Create a `PaymentFraudAnalysisConsumer` — consume and log events from `payment-events` | ☐ |
| Configure manual acknowledgment (`enable-auto-commit: false`) | ☐ |
| Test: produce 10 events, verify all are consumed in order | ☐ |
| Configure Dead Letter Topic (DLT): throw exception in consumer → verify event goes to DLT | ☐ |
| Implement Outbox Pattern: save payment + insert outbox event in one transaction | ☐ |
| Research Q1, Q7 answered | ☐ |

---

## Week 2: CQRS + Event Sourcing

| Task | Done |
|------|------|
| Implement `PaymentCommandService` (write side) with event publishing | ☐ |
| Create a separate `PaymentReadModel` table (denormalized, query-optimized) | ☐ |
| Implement Kafka consumer that updates read model on each payment event | ☐ |
| Test CQRS: submit payment command → verify read model updates (with < 1s delay) | ☐ |
| Implement Event Sourcing: `payment_events` table, store all events instead of state | ☐ |
| Implement `PaymentAggregateService.reconstruct()`: replay events to get current state | ☐ |
| Implement a projection: build read model by replaying events from the beginning | ☐ |
| Research Q2, Q3 answered | ☐ |

---

## Week 3: Saga Pattern

| Task | Done |
|------|------|
| Implement choreography saga (2-step): payment-created → fraud-check → payment-captured | ☐ |
| Implement compensating transaction: if step 2 fails → reverse step 1 | ☐ |
| Implement orchestration saga for payment dispute (5-step with AI) | ☐ |
| Implement idempotency check: `processed_events` table, skip duplicate events | ☐ |
| Test idempotency: publish the same event twice → verify processed only once | ☐ |
| Simulate network failure in step 3 of saga → verify compensation runs correctly | ☐ |
| Research Q4, Q5 answered | ☐ |

---

## Week 4: Advanced Patterns

| Task | Done |
|------|------|
| Add Resilience4j circuit breaker around LLM API calls | ☐ |
| Add retry with exponential backoff (3 retries, 2s initial) | ☐ |
| Add bulkhead: max 10 concurrent LLM calls | ☐ |
| Test: simulate 100% LLM failure → circuit opens → fallback triggers | ☐ |
| Design Kafka topic strategy: topics, partition count, message key for your payment system | ☐ |
| Implement rolling deployment test: old + new consumer version running simultaneously | ☐ |
| Research Q6, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build an event-driven payment system with Kafka, CQRS read/write separation, and the Saga pattern? Can you explain CAP, eventual consistency, and idempotency? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| Kafka producer + consumer with Spring | |
| Dead Letter Topic and error handling | |
| Outbox Pattern | |
| CQRS (command side + read side + sync via Kafka) | |
| Event Sourcing (events table + reconstruction) | |
| Saga pattern (choreography + orchestration) | |
| Compensating transactions | |
| Kafka idempotency | |
| Resilience4j (circuit breaker, retry, bulkhead) | |
| CAP theorem and eventual consistency | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Kafka | | |
| Week 2 — CQRS + Event Sourcing | | |
| Week 3 — Saga Pattern | | |
| Week 4 — Advanced Patterns | | |
| **Total** | | |
