# Module 05: LangChain4j + Agentic Workflows
**Duration:** 3 weeks (2 hours/day)
**Goal:** Master LangChain4j as an alternative to Spring AI for complex agentic workflows, multi-step reasoning, and memory-enabled agents.

---

## The Big Idea

```
Spring AI:    Official Spring project. Best for: Spring Boot integration,
              quick setup, standard RAG, simple agents.
              Follows Spring conventions you already know.

LangChain4j:  Independent library. Best for: complex multi-step agents,
              rich memory systems, advanced tool use, complex pipelines.
              Closer to Python LangChain concepts.

2025 reality:
  - Use Spring AI as your PRIMARY framework
  - Learn LangChain4j for ADVANCED AGENTIC patterns Spring AI lacks
  - Many enterprise AI systems use both (Spring AI for API/RAG,
    LangChain4j for complex agent workflows)

LangChain4j unique strengths:
  ✅ AI Services: interfaces annotated with @AiService (cleaner API)
  ✅ Persistent memory: ChatMemoryStore (SQLite, PostgreSQL backends)
  ✅ Streaming with more control
  ✅ Tool execution with automatic retry
  ✅ RAG with more pipeline control
```

---

## Week 1: LangChain4j Fundamentals

**Monday — Setup and First Chat**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>0.35.0</version>
</dependency>
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
    <version>0.35.0</version>
</dependency>
```

```yaml
# application.yml
langchain4j:
  open-ai:
    chat-model:
      api-key: ${OPENAI_API_KEY}
      model-name: gpt-4o
      temperature: 0.1
      max-tokens: 2000
  # For local Ollama:
  ollama:
    chat-model:
      base-url: http://localhost:11434
      model-name: llama3.1
```

```java
// LangChain4j AI Service: the cleanest way to define AI behavior
// Annotate an interface → LangChain4j implements it automatically

@AiService
public interface PaymentAssistant {

    @SystemMessage("""
        You are a payment support specialist.
        Answer questions about payments professionally and accurately.
        Never reveal card numbers or sensitive data.
        """)
    String chat(@UserMessage String userMessage);

    @SystemMessage("Classify the payment issue. Return ONLY: BILLING, TECHNICAL, or FRAUD.")
    String classify(@UserMessage String message);
}

// Spring Boot auto-wires the implementation:
@RestController
public class SupportController {

    @Autowired
    private PaymentAssistant paymentAssistant;

    @PostMapping("/chat")
    public String chat(@RequestBody String message) {
        return paymentAssistant.chat(message);
    }
}
```

**Tuesday — Structured Output with LangChain4j**
```java
// LangChain4j structured output: return Java records directly

record PaymentExtraction(
    @Description("The payment ID in format PAY-XXXXX") String paymentId,
    @Description("Issue type: DECLINED, FAILED, PENDING, REFUND_REQUEST") String issueType,
    @Description("Amount in the currency specified, null if not mentioned") Double amount,
    @Description("Currency code (USD, EUR, GBP), null if not mentioned") String currency,
    @Description("True if customer used words like urgent, ASAP, critical") boolean isUrgent
) {}

// AI Service returning structured output:
@AiService
public interface DataExtractionService {

    @SystemMessage("""
        Extract payment information from the customer message.
        If a field is not mentioned, use null for strings and false for booleans.
        """)
    PaymentExtraction extractPaymentInfo(@UserMessage String customerMessage);

    @SystemMessage("Extract ALL payment IDs mentioned in the text.")
    List<String> extractPaymentIds(@UserMessage String text);
}

// Usage:
PaymentExtraction extracted = dataExtractionService
    .extractPaymentInfo("My payment PAY-9981 for £450 failed urgently!");
// extracted.paymentId() == "PAY-9981"
// extracted.amount() == 450.0
// extracted.isUrgent() == true
```

**Wednesday–Thursday — Tools in LangChain4j**
```java
// LangChain4j tools: annotate methods with @Tool

@Component
public class PaymentTools {

    private final PaymentRepository paymentRepository;

    @Tool("Get payment details by payment ID. Returns status, amount, currency, and failure reason.")
    public String getPaymentDetails(
            @P("Payment ID in format PAY-XXXXX") String paymentId) {

        return paymentRepository.findById(paymentId)
            .map(p -> String.format(
                "Payment %s: status=%s, amount=%.2f %s, failure_reason=%s",
                p.getId(), p.getStatus(), p.getAmount(), p.getCurrency(), p.getFailureReason()
            ))
            .orElse("Payment " + paymentId + " not found");
    }

    @Tool("Search recent payments for a customer. Returns list of payment summaries.")
    public List<String> searchCustomerPayments(
            @P("Customer ID") String customerId,
            @P("Maximum results to return (1-20)") int limit) {

        return paymentRepository.findByCustomerId(customerId, PageRequest.of(0, limit))
            .stream()
            .map(p -> String.format("%s: %s (%.2f %s)", p.getId(), p.getStatus(), p.getAmount(), p.getCurrency()))
            .collect(Collectors.toList());
    }

    @Tool("Check if a payment is eligible for a refund. Returns eligibility and reason.")
    public String checkRefundEligibility(@P("Payment ID") String paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow(() -> new RuntimeException("Payment not found"));

        boolean eligible = payment.getStatus() == PaymentStatus.COMPLETED
            && payment.getCreatedAt().isAfter(LocalDateTime.now().minusDays(30));

        return eligible
            ? "Eligible for refund: payment is within 30-day window"
            : "Not eligible: " + (payment.getStatus() != PaymentStatus.COMPLETED
                ? "payment was not successful"
                : "payment is older than 30 days");
    }
}

// Attach tools to AI Service:
@AiService
@Slf4j
public interface PaymentAgent {

    @SystemMessage("""
        You are a payment support agent with access to the payment database.
        Use the available tools to look up real payment information.
        Always verify payment details before answering.
        """)
    String handleQuery(@UserMessage String query,
                       @V("customerId") String customerId);
}

// In configuration:
@Bean
PaymentAgent paymentAgent(ChatLanguageModel chatModel, PaymentTools tools) {
    return AiServices.builder(PaymentAgent.class)
        .chatLanguageModel(chatModel)
        .tools(tools)
        .build();
}
```

**Friday — Streaming with LangChain4j**
```java
// LangChain4j streaming: real-time token output

@AiService
public interface StreamingPaymentAssistant {

    @SystemMessage("You are a payment support specialist.")
    TokenStream chatStream(@UserMessage String message);
}

// Spring WebFlux SSE controller:
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return Flux.create(sink -> {
        streamingAssistant.chatStream(message)
            .onNext(token -> sink.next(token))
            .onComplete(response -> sink.complete())
            .onError(sink::error)
            .start();
    });
}
```

---

## Week 2: Memory-Enabled Agents

**Monday–Tuesday — Chat Memory**
```java
// LangChain4j persistent memory: agent remembers conversation across requests

// In-memory (development only):
ChatMemory memory = MessageWindowChatMemory.withMaxMessages(20);

// Persistent (production):
// Store conversations in PostgreSQL
public class PostgresChatMemoryStore implements ChatMemoryStore {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        return jdbcTemplate.query(
            "SELECT message_type, content FROM chat_memory WHERE session_id = ? ORDER BY created_at",
            (rs, row) -> {
                String type = rs.getString("message_type");
                String content = rs.getString("content");
                return switch (type) {
                    case "USER"      -> new UserMessage(content);
                    case "ASSISTANT" -> new AiMessage(content);
                    case "SYSTEM"    -> new SystemMessage(content);
                    default -> throw new IllegalArgumentException("Unknown type: " + type);
                };
            },
            memoryId.toString()
        );
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        jdbcTemplate.update("DELETE FROM chat_memory WHERE session_id = ?", memoryId.toString());
        for (ChatMessage msg : messages) {
            jdbcTemplate.update(
                "INSERT INTO chat_memory (session_id, message_type, content, created_at) VALUES (?, ?, ?, NOW())",
                memoryId.toString(),
                msg.getClass().getSimpleName().toUpperCase().replace("MESSAGE", ""),
                ((TextContent) msg.contents().get(0)).text()
            );
        }
    }

    @Override
    public void deleteMessages(Object memoryId) {
        jdbcTemplate.update("DELETE FROM chat_memory WHERE session_id = ?", memoryId.toString());
    }
}

// AI Service with persistent memory:
@Bean
PaymentAssistant paymentAssistantWithMemory(
        ChatLanguageModel model,
        PostgresChatMemoryStore memoryStore) {

    return AiServices.builder(PaymentAssistant.class)
        .chatLanguageModel(model)
        .chatMemoryProvider(memoryId ->
            MessageWindowChatMemory.builder()
                .id(memoryId)
                .maxMessages(20)
                .chatMemoryStore(memoryStore)
                .build()
        )
        .build();
}
```

**Wednesday — Memory Strategies**
```
LangChain4j memory strategies:

1. MessageWindowChatMemory (simplest):
   Keep last N messages. Simple, predictable.
   Risk: important context from early in conversation may be lost.

2. TokenWindowChatMemory:
   Keep messages until token budget is reached.
   Better: never exceeds context window.

3. Summarizing memory (advanced):
   When memory is full, summarize old messages and keep the summary.
   Best for long conversations — context preserved but compressed.
   Implementation: when message count > threshold, call LLM to summarize
   old messages → replace them with the summary.

4. Entity memory (domain-specific):
   Extract key entities (payment ID, customer ID) from the conversation
   and always keep them in context, regardless of which messages are dropped.
   Best for: customer support where the payment ID must never be lost.

// Summarizing memory:
public void summarizeOldMessages(List<ChatMessage> messages, int keepRecent) {
    if (messages.size() <= keepRecent) return;

    List<ChatMessage> toSummarize = messages.subList(0, messages.size() - keepRecent);

    String summary = summarizerLLM.prompt()
        .system("Summarize this conversation history into 3-5 sentences, preserving key facts.")
        .user(formatMessages(toSummarize))
        .call()
        .content();

    // Replace old messages with summary
    messages.subList(0, messages.size() - keepRecent).clear();
    messages.add(0, new SystemMessage("Conversation summary: " + summary));
}
```

**Thursday–Friday — RAG with LangChain4j**
```java
// LangChain4j RAG pipeline

// 1. Setup embedding model + vector store:
@Bean
EmbeddingModel embeddingModel() {
    return OpenAiEmbeddingModel.builder()
        .apiKey(openAiApiKey)
        .modelName("text-embedding-3-small")
        .build();
}

@Bean
EmbeddingStore<TextSegment> embeddingStore() {
    return PgVectorEmbeddingStore.builder()
        .host("localhost")
        .port(5432)
        .database("payments")
        .user("postgres")
        .password(dbPassword)
        .table("document_embeddings")
        .dimension(1536)
        .build();
}

// 2. Ingest documents:
@Service
public class LangChain4jIngestionService {

    private final EmbeddingModel embeddingModel;
    private final EmbeddingStore<TextSegment> embeddingStore;

    public void ingest(String text, Map<String, String> metadata) {
        DocumentSplitter splitter = DocumentSplitters.recursive(500, 100);

        Document document = Document.from(text, Metadata.from(metadata));
        List<TextSegment> segments = splitter.split(document);

        List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
        embeddingStore.addAll(embeddings, segments);
    }
}

// 3. AI Service with RAG (ContentRetriever):
@Bean
ContentRetriever contentRetriever(
        EmbeddingModel model,
        EmbeddingStore<TextSegment> store) {

    return EmbeddingStoreContentRetriever.builder()
        .embeddingModel(model)
        .embeddingStore(store)
        .maxResults(5)
        .minScore(0.65)
        .build();
}

@Bean
PaymentPolicyAssistant policyAssistant(
        ChatLanguageModel chatModel,
        ContentRetriever contentRetriever) {

    return AiServices.builder(PaymentPolicyAssistant.class)
        .chatLanguageModel(chatModel)
        .contentRetriever(contentRetriever)  // RAG auto-wired
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .build();
}

// AI Service interface:
@AiService
public interface PaymentPolicyAssistant {
    @SystemMessage("""
        You are a payment policy expert. Answer questions ONLY using the retrieved documents.
        If the documents don't contain the answer, say "I don't have that information."
        """)
    String askAboutPolicy(@UserMessage String question);
}
```

---

## Week 3: Complex Agentic Workflows

**Monday–Wednesday — Chaining Agents**
```java
// Chain: each agent's output feeds the next

@Service
public class PaymentDisputeWorkflow {

    private final DataExtractionService extractionAgent;
    private final RiskAssessmentAgent riskAgent;
    private final PolicyLookupAgent policyAgent;
    private final ResponseGeneratorAgent responseAgent;

    public DisputeResolution processDispute(String customerMessage) {

        // Step 1: Extract structured data from customer message
        DisputeInfo disputeInfo = extractionAgent.extractDisputeInfo(customerMessage);

        // Step 2: Assess risk based on extracted data
        RiskAssessment risk = riskAgent.assessRisk(
            disputeInfo.paymentId(),
            disputeInfo.amount(),
            disputeInfo.reason()
        );

        // Step 3: Look up relevant policies
        String relevantPolicies = policyAgent.findRelevantPolicies(
            disputeInfo.reason(),
            risk.riskLevel()
        );

        // Step 4: Generate customer-facing response
        String response = responseAgent.generateResponse(
            customerMessage,
            disputeInfo.toString(),
            risk.toString(),
            relevantPolicies
        );

        return new DisputeResolution(
            disputeInfo.paymentId(),
            risk.riskLevel(),
            risk.recommendation(),
            response
        );
    }
}
```

**Thursday — Tool Use with Retry and Error Handling**
```java
// LangChain4j handles tool errors gracefully:
// If a tool throws an exception, the LLM sees the error message
// and can decide to retry with different parameters or give up

@Component
public class ResilientPaymentTools {

    @Tool("Get payment status. If payment not found, returns a clear error message.")
    public String getPaymentStatus(@P("Payment ID") String paymentId) {
        try {
            Payment payment = paymentRepository.findById(paymentId)
                .orElseThrow(() -> new RuntimeException("Payment not found: " + paymentId));
            return "Status: " + payment.getStatus() + ", Amount: " + payment.getAmount();
        } catch (Exception e) {
            // LangChain4j passes this message to the LLM
            // The LLM can then decide what to do next
            return "Error: " + e.getMessage() + ". Please verify the payment ID format (PAY-XXXXX).";
        }
    }
}

// LangChain4j automatic retry for tools:
@Bean
PaymentAgent paymentAgentWithRetry(ChatLanguageModel model, ResilientPaymentTools tools) {
    return AiServices.builder(PaymentAgent.class)
        .chatLanguageModel(model)
        .tools(tools)
        .build();
    // LangChain4j automatically handles the error→retry loop
}
```

**Friday — Comparison: Spring AI vs LangChain4j**
```
Feature                     Spring AI           LangChain4j
────────────────────────────────────────────────────────────
Spring Boot integration     Native              Via starters
AI Service interface        No                  Yes (@AiService)
Chat memory                 Basic               Rich (multiple backends)
Persistent memory           Manual              ChatMemoryStore interface
MCP support                 Yes (v1.0)          Limited
Tool use                    @Tool               @Tool (different annotation)
RAG                         QuestionAnswerAdvisor EmbeddingStoreRetriever
Streaming                   Flux<String>        TokenStream
Vector stores               Many (pgvector, Qdrant) Many (pgvector, Qdrant)
Embedding models            Spring AI models    LangChain4j models
Documentation               Spring docs         langchain4j.dev

When to use Spring AI:
  ✅ Spring Boot app (primary choice)
  ✅ Quick setup, standard patterns
  ✅ MCP server/client
  ✅ Spring Security integration

When to use LangChain4j:
  ✅ Complex agentic workflows
  ✅ Rich memory management (persistent, summarizing)
  ✅ @AiService interface pattern (very clean API)
  ✅ Fine-grained control over the RAG pipeline

Both together:
  ✅ Spring AI for HTTP layer, security, MCP
  ✅ LangChain4j for complex agent logic and memory
```
