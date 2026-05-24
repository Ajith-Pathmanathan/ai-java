# Module 07: Distributed Systems + Event-Driven Architecture
**Duration:** 4 weeks (2 hours/day)
**Goal:** Master Kafka, CQRS, Event Sourcing, Saga pattern, and service mesh — the distributed systems architecture every senior Java engineer must know.

---

## The Big Idea

```
Single-service apps are simple. But in production:
  - One service handling everything becomes a bottleneck
  - A single bug takes down the entire system
  - You can't scale just the slow part

Distributed systems solve this by splitting responsibilities:
  - Services communicate via events (Kafka)
  - Each service owns its data (no shared database)
  - Failures are isolated (one service fails, others keep running)

For AI systems specifically:
  - AI inference is slow (2-5 seconds) → don't block the main service
  - Process AI tasks asynchronously via Kafka
  - CQRS separates reads (AI queries) from writes (transactions)
  - Saga pattern coordinates multi-step processes that involve AI agents
```

---

## Week 1: Apache Kafka

**Monday — Kafka Core Concepts**
```java
// Kafka: a distributed event streaming platform
// Key concepts:

// Topic: a category/feed of events (like a database table, but for events)
// Partition: a topic is split into partitions (enables parallelism)
// Offset: position of an event in a partition (immutable, sequential)
// Consumer Group: a group of consumers that share topic processing
// Producer: sends events to a topic
// Consumer: reads events from a topic

// Why Kafka for Java/AI systems:
// ✅ Decouple slow AI processing from fast HTTP responses
// ✅ Retry failed AI tasks without losing events
// ✅ Fan-out: one payment event → multiple AI consumers
// ✅ Durability: events stored for days/weeks (replay-able)
// ✅ Ordering: events in a partition are strictly ordered

// Spring Kafka dependency:
// <dependency>
//   <groupId>org.springframework.kafka</groupId>
//   <artifactId>spring-kafka</artifactId>
// </dependency>
```

**Tuesday — Producing and Consuming Events**
```java
// Producer: send payment events to Kafka
@Service
@Slf4j
public class PaymentEventProducer {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    public void publishPaymentCreated(Payment payment) {
        PaymentEvent event = new PaymentEvent(
            payment.getId(),
            payment.getCustomerId(),
            payment.getAmount(),
            payment.getCurrency(),
            PaymentEventType.CREATED,
            Instant.now()
        );

        kafkaTemplate.send("payment-events", payment.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish payment event: {}", payment.getId(), ex);
                } else {
                    log.info("Published payment event: {} offset={}",
                        payment.getId(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}

// Consumer: process payment events and run AI analysis
@Component
@Slf4j
public class PaymentFraudAnalysisConsumer {

    private final FraudDetectionAgent fraudAgent;
    private final FraudAlertService alertService;

    @KafkaListener(
        topics = "payment-events",
        groupId = "fraud-detection-group",
        containerFactory = "paymentEventListenerFactory"
    )
    public void analyzePayment(PaymentEvent event, Acknowledgment ack) {
        if (event.type() != PaymentEventType.CREATED) {
            ack.acknowledge();
            return;
        }

        try {
            // AI fraud analysis (this can take 1-3 seconds — fine in async consumer)
            FraudAssessment assessment = fraudAgent.analyze(event);

            if (assessment.riskLevel() == RiskLevel.HIGH) {
                alertService.sendFraudAlert(event.paymentId(), assessment);
            }

            ack.acknowledge();  // commit offset only on success

        } catch (Exception e) {
            log.error("Failed to analyze payment: {}", event.paymentId(), e);
            // DON'T acknowledge → Kafka will redeliver → retry
            // After max retries → dead letter topic
        }
    }
}
```

**Wednesday — Kafka Configuration and Reliability**
```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all              # wait for all replicas to confirm
      retries: 3
      enable-idempotence: true  # prevent duplicate events on retry
    consumer:
      group-id: payment-service
      auto-offset-reset: earliest   # on first start, read from beginning
      enable-auto-commit: false     # manual acknowledgment (safer)
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

```java
// Dead Letter Topic: catch events that fail after max retries
@Configuration
public class KafkaConfiguration {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // After 3 retries with 2-second backoff, send to dead letter topic
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );

        FixedBackOff backOff = new FixedBackOff(2000L, 3L);
        return new DefaultErrorHandler(recoverer, backOff);
    }
}

// Dead letter consumer: alert and investigate failed events
@KafkaListener(topics = "payment-events.DLT", groupId = "dlt-handler")
public void handleDeadLetter(ConsumerRecord<String, PaymentEvent> record) {
    log.error("Dead letter event: topic={} key={} value={}",
        record.topic(), record.key(), record.value());
    alertService.sendDeadLetterAlert(record.key(), record.value());
}
```

**Thursday–Friday — Kafka Patterns for AI**
```java
// Pattern: async AI processing without blocking HTTP response

// 1. HTTP request comes in → save to DB → publish to Kafka → return 202 Accepted
@PostMapping("/payments/{id}/analyze")
public ResponseEntity<Void> triggerAnalysis(@PathVariable String id) {
    Payment payment = paymentRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("Payment " + id));

    producer.publishPaymentCreated(payment);

    return ResponseEntity.accepted().build();  // 202 = processing async
}

// 2. Kafka consumer runs AI analysis → stores result in DB
@KafkaListener(topics = "payment-events")
public void runAiAnalysis(PaymentEvent event) {
    FraudAssessment result = aiAgent.analyze(event);
    assessmentRepository.save(result);
    // Frontend polls GET /payments/{id}/analysis for the result
}

// Pattern: fan-out — one event → multiple processors
// Topic: payment-events
// Consumer group A: fraud-detection (AI fraud analysis)
// Consumer group B: compliance-logger (log for regulatory purposes)
// Consumer group C: notification-service (send payment confirmation)
// Each group processes independently, each at its own pace
```

---

## Week 2: CQRS + Event Sourcing

**Monday–Tuesday — CQRS Pattern**
```java
// CQRS: Command Query Responsibility Segregation
// Split your application into:
//   Write side (Commands): handle mutations (create, update, delete)
//   Read side (Queries):  handle reads (search, report, display)

// Why CQRS for AI systems:
// - Reads and writes have very different scaling requirements
// - AI reads (RAG queries) are expensive → cache aggressively on read side
// - Writes (payment created) trigger AI processing via events
// - Read side can have AI-optimized data models (vector embeddings, search indexes)

// Command side: handle mutations
@Service
public class PaymentCommandService {

    private final PaymentRepository paymentRepository;
    private final PaymentEventProducer eventProducer;

    @Transactional
    public Payment createPayment(CreatePaymentCommand cmd) {
        Payment payment = new Payment(
            UUID.randomUUID(),
            cmd.customerId(),
            cmd.amount(),
            cmd.currency(),
            PaymentStatus.PENDING
        );

        Payment saved = paymentRepository.save(payment);

        // Publish event → triggers read side update + AI analysis
        eventProducer.publishPaymentCreated(saved);

        return saved;
    }
}

// Query side: optimized for reads (different data model, can be separate DB)
@Service
public class PaymentQueryService {

    private final PaymentSearchRepository searchRepository;  // Elasticsearch or PG view

    public List<PaymentSummary> searchPayments(PaymentSearchQuery query) {
        return searchRepository.findByCustomerAndStatus(
            query.customerId(),
            query.status(),
            query.fromDate(),
            query.toDate()
        );
    }

    public AiAnalysisSummary getAiAnalysis(String paymentId) {
        return searchRepository.findAiAnalysisById(paymentId);
    }
}

// Query controller (read-only, no mutations):
@RestController
@RequestMapping("/api/payments")
public class PaymentQueryController {

    @GetMapping("/{id}")
    public PaymentDetail getPayment(@PathVariable String id) {
        return queryService.getPaymentDetail(id);
    }

    @GetMapping("/search")
    public List<PaymentSummary> searchPayments(PaymentSearchQuery query) {
        return queryService.searchPayments(query);
    }
}
```

**Wednesday–Thursday — Event Sourcing**
```java
// Event Sourcing: store EVENTS, not state
// State = replay of all events from the beginning

// Instead of:
//   UPDATE payments SET status='COMPLETED' WHERE id='PAY-123'

// You store:
//   INSERT INTO payment_events (payment_id, event_type, data, timestamp)
//   VALUES ('PAY-123', 'PAYMENT_COMPLETED', {...}, NOW())

// The current state is derived by replaying events:

@Entity
@Table(name = "payment_events")
public class PaymentEvent {
    @Id private UUID id;
    private String paymentId;
    private String eventType;      // CREATED, AUTHORIZED, CAPTURED, REFUNDED, FAILED
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> data;
    private Instant occurredAt;
    private Long version;          // optimistic locking
}

// Reconstruct current payment state from events:
@Service
public class PaymentAggregateService {

    public Payment reconstruct(String paymentId) {
        List<PaymentEvent> events = paymentEventRepository
            .findByPaymentIdOrderByVersionAsc(paymentId);

        if (events.isEmpty()) {
            throw new NotFoundException("Payment " + paymentId + " not found");
        }

        // Apply events one by one to build current state
        Payment payment = new Payment();
        for (PaymentEvent event : events) {
            payment = applyEvent(payment, event);
        }
        return payment;
    }

    private Payment applyEvent(Payment current, PaymentEvent event) {
        return switch (event.eventType()) {
            case "CREATED"    -> new Payment(event.data());
            case "AUTHORIZED" -> current.withStatus(AUTHORIZED);
            case "CAPTURED"   -> current.withStatus(COMPLETED).withCapturedAt(event.occurredAt());
            case "REFUNDED"   -> current.withStatus(REFUNDED);
            case "FAILED"     -> current.withStatus(FAILED).withFailureReason(
                                    (String) event.data().get("reason"));
            default -> current;
        };
    }
}

// Benefits of Event Sourcing for AI:
// - Complete audit trail: every state change is recorded
// - Time travel: reconstruct payment state at any point in time
// - AI training data: rich event history is perfect for training ML models
// - GDPR right to erasure: mark events as deleted (soft delete, not physical delete)
```

**Friday — Projections and Read Models**
```java
// Event sourcing creates a problem: querying events is slow
// Solution: Projections — build optimized read models from events

@Component
public class PaymentReadModelProjector {

    @EventHandler(event = PaymentEvent.class)
    public void on(PaymentEvent event) {
        switch (event.eventType()) {
            case "CREATED" -> createReadModel(event);
            case "CAPTURED" -> updateReadModelStatus(event.paymentId(), COMPLETED);
            case "REFUNDED" -> updateReadModelStatus(event.paymentId(), REFUNDED);
            case "FAILED"   -> updateReadModelWithFailure(event);
        }
    }

    private void createReadModel(PaymentEvent event) {
        PaymentReadModel model = new PaymentReadModel(
            event.paymentId(),
            (String) event.data().get("customerId"),
            ((Number) event.data().get("amount")).doubleValue(),
            (String) event.data().get("currency"),
            PENDING,
            event.occurredAt()
        );
        paymentReadModelRepository.save(model);
    }
}
```

---

## Week 3: Saga Pattern + Distributed Transactions

**Monday–Tuesday — The Distributed Transaction Problem**
```
Problem: a payment involves multiple services:
  PaymentService → FraudCheckService → NotificationService → LedgerService

What if FraudCheckService approves but LedgerService fails?
  - Payment is approved but not recorded in the ledger
  - Customer is charged but the merchant is never paid
  - Disaster

Traditional solution: 2-Phase Commit (2PC)
  All services prepare → coordinator confirms → all services commit
  Problem: blocking, slow, doesn't work with microservices

Better solution: SAGA PATTERN
  Each step is a local transaction with a compensating transaction
  If step N fails → run compensating transactions for steps 1..N-1
```

**Wednesday — Choreography Saga**
```java
// Choreography saga: services communicate via events, no central coordinator
// Each service listens to events and publishes its own events

// Step 1: Payment service creates payment, publishes PaymentCreated
@KafkaListener(topics = "payment-created")
// → FraudCheckService listens

// Step 2: FraudCheck runs, publishes FraudCheckPassed or FraudCheckFailed
@KafkaListener(topics = "fraud-check-passed")
// → PaymentCapture listens

// Step 3: PaymentCapture runs, publishes PaymentCaptured or PaymentCaptureFailed
@KafkaListener(topics = "payment-capture-failed")
// → triggers compensation

// Compensation: if capture fails → refund any held funds → notify customer
@KafkaListener(topics = "payment-capture-failed")
public void compensateFailedCapture(PaymentCaptureFailedEvent event) {
    paymentService.releaseHeldFunds(event.paymentId());
    notificationService.sendPaymentFailedEmail(event.customerId());
    publisher.publish("payment-failed", new PaymentFailedEvent(event.paymentId()));
}
```

**Thursday — Orchestration Saga**
```java
// Orchestration saga: central coordinator manages the workflow
// Cleaner for complex multi-step processes

@Service
public class PaymentSagaOrchestrator {

    public void executePaymentSaga(String paymentId) {
        try {
            // Step 1: Fraud check
            FraudCheckResult fraud = fraudCheckService.check(paymentId);
            if (fraud.isRejected()) {
                paymentService.markFraudRejected(paymentId, fraud.reason());
                notifyCustomerFraudRejection(paymentId);
                return;  // saga complete (rejected path)
            }

            // Step 2: Authorize payment
            AuthorizationResult auth = bankService.authorize(paymentId);
            if (!auth.isAuthorized()) {
                paymentService.markDeclined(paymentId, auth.declineCode());
                notifyCustomerDeclined(paymentId);
                return;  // saga complete (declined path)
            }

            // Step 3: Capture funds
            try {
                captureService.capture(paymentId);
            } catch (CaptureException e) {
                // Compensate: release authorization
                bankService.releaseAuthorization(auth.authorizationId());
                paymentService.markFailed(paymentId, e.getMessage());
                notifyCustomerFailed(paymentId);
                return;
            }

            // Step 4: Record in ledger
            ledgerService.record(paymentId);

            // Step 5: Notify success
            notifyCustomerSuccess(paymentId);
            paymentService.markCompleted(paymentId);

        } catch (Exception e) {
            log.error("Saga failed for payment {}", paymentId, e);
            paymentService.markFailed(paymentId, "Saga execution failed");
        }
    }
}
```

**Friday — Saga for AI Agent Workflows**
```java
// Saga for long-running AI agent tasks:
// Dispute resolution: customer submits → merchant 48h to respond → AI decision

@Service
public class DisputeSagaOrchestrator {

    // Step 1: Receive dispute
    @KafkaListener(topics = "dispute-submitted")
    public void onDisputeSubmitted(DisputeEvent event) {
        Dispute dispute = new Dispute(event.paymentId(), event.reason(), PENDING);
        disputeRepository.save(dispute);

        // Notify merchant (async email)
        notificationService.sendMerchantDisputeNotification(event.merchantId(), dispute);

        // Schedule: check for merchant response after 48h
        scheduler.schedule(
            () -> onMerchantResponseDeadline(dispute.getId()),
            48, TimeUnit.HOURS
        );
    }

    // Step 2: Merchant responds (or deadline expires)
    public void onMerchantResponseDeadline(String disputeId) {
        Dispute dispute = disputeRepository.findById(disputeId).orElseThrow();

        if (dispute.getMerchantResponse() == null) {
            // No response → auto-resolve in favor of customer
            resolveDispute(disputeId, "CUSTOMER_WIN", "Merchant did not respond within 48 hours");
        } else {
            // AI analyzes both sides
            DisputeAnalysis analysis = aiAgent.analyzeDispute(
                dispute.getCustomerReason(),
                dispute.getMerchantResponse()
            );
            resolveDispute(disputeId, analysis.decision(), analysis.reasoning());
        }
    }

    private void resolveDispute(String disputeId, String decision, String reason) {
        disputeRepository.updateResolution(disputeId, decision, reason);

        if ("CUSTOMER_WIN".equals(decision)) {
            refundService.initiateRefund(disputeId);
        }

        notificationService.sendDisputeResolution(disputeId, decision);
    }
}
```

---

## Week 4: Service Mesh + Advanced Patterns

**Monday–Wednesday — Resilience Patterns**
```java
// Retry, Circuit Breaker, Bulkhead with Resilience4j

@Configuration
public class ResilienceConfiguration {

    @Bean
    public CircuitBreakerConfig llmCircuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)           // open after 50% failure rate
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(10)
            .permittedNumberOfCallsInHalfOpenState(2)
            .build();
    }

    @Bean
    public RetryConfig llmRetryConfig() {
        return RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(2))
            .retryOnException(e -> e instanceof OpenAiApiException)
            .build();
    }

    // Bulkhead: limit concurrent calls to LLM (prevent thread exhaustion)
    @Bean
    public BulkheadConfig llmBulkheadConfig() {
        return BulkheadConfig.custom()
            .maxConcurrentCalls(20)     // max 20 concurrent LLM calls
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
    }
}

// Apply all three:
@CircuitBreaker(name = "llm-api", fallbackMethod = "fallback")
@Retry(name = "llm-api")
@Bulkhead(name = "llm-api")
public String callLlm(String prompt) {
    return chatClient.prompt().user(prompt).call().content();
}
```

**Thursday–Friday — Key Distributed Systems Concepts**
```
CAP Theorem (exam question at every architect interview):
  C = Consistency: every read gets the most recent write
  A = Availability: every request gets a response
  P = Partition tolerance: system works even if nodes can't communicate

  You CANNOT have all three simultaneously in a distributed system.
  Choose two: CP, AP, or CA (CA is only for non-distributed systems)

  PostgreSQL = CP (consistent + partition tolerant, may reject requests during partition)
  Cassandra  = AP (always available, may return stale data during partition)
  Kafka      = AP (always available, events are ordered within partition)

  For fintech AI:
    - Payment data → CP (must be consistent — no double charges)
    - Chat history → AP (availability > consistency for support chats)
    - Embeddings   → AP (ok to serve slightly stale search results)

Eventual Consistency in CQRS:
  Write side updates → publishes event → read side updates ASYNCHRONOUSLY
  There is a lag (usually milliseconds to seconds)
  During this lag: a user might see a slightly stale view

  Example: customer pays → PaymentCreated event → read model updates 500ms later
  During those 500ms: customer sees "Payment pending" even if it completed
  Solution: return the command result directly (bypass the read model for just-written data)

Idempotency in Kafka:
  Consumers may receive the same event twice (at-least-once delivery)
  Every consumer must be idempotent: processing the same event twice = same result
  Implementation: check if event has already been processed before acting
  Use a processed_events table with the event ID as the primary key
```
