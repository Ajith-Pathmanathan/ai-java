# Module 03: RAG Systems with Spring AI
**Duration:** 4 weeks (2 hours/day)
**Goal:** Build a production-grade RAG pipeline in Java/Spring Boot that answers questions from your own documents.

---

## The Big Idea

```
RAG = Retrieval Augmented Generation

WITHOUT RAG:
  User: "What is our refund policy for fraudulent transactions?"
  LLM: "I don't know your specific policy" OR hallucinates an answer

WITH RAG:
  1. RETRIEVE: search vector DB → find relevant policy chunks
  2. AUGMENT:  inject those chunks into the LLM prompt
  3. GENERATE: LLM answers based on YOUR documents, not general training

Result: LLM gives accurate, grounded answers from your actual policy documents.
        And it cites sources.

This is the #1 AI pattern used in enterprise software in 2025.
Every company with documents (policies, manuals, knowledge bases)
is building a RAG system right now.
```

---

## Week 1: RAG Architecture and Retrieval

**Monday — The RAG Pipeline**
```
RAG has two phases:

INGESTION (offline, run once or periodically):
  Documents → Read → Split (chunk) → Embed → Store in Vector DB

QUERY (online, runs every user request):
  User Query → Embed Query → Vector Search → Retrieve Top-K Chunks
  → Build Prompt (query + chunks) → LLM → Answer

The ingestion pipeline is run when you add/update documents.
The query pipeline runs in real-time for every user question.

Components in Spring AI:
  DocumentReader:   reads PDFs, text files, web pages
  TokenTextSplitter: splits into chunks
  EmbeddingModel:   converts text to vectors
  VectorStore:      stores and searches embeddings (pgvector, Qdrant)
  ChatClient:       generates the final answer
```

**Tuesday–Wednesday — Naive RAG Implementation**
```java
// Naive RAG: simplest implementation, works for 70% of use cases

@Service
public class NaiveRagService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    public String query(String userQuestion) {
        // Step 1: Retrieve relevant documents
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(userQuestion)
                .withTopK(5)
                .withSimilarityThreshold(0.65)
        );

        if (relevantDocs.isEmpty()) {
            return "I don't have information about that in our knowledge base.";
        }

        // Step 2: Build context from retrieved documents
        String context = relevantDocs.stream()
            .map(doc -> String.format(
                "Source: %s\n%s",
                doc.getMetadata().get("source"),
                doc.getContent()
            ))
            .collect(Collectors.joining("\n\n---\n\n"));

        // Step 3: Build the RAG prompt
        String systemPrompt = """
            You are a payment policy assistant.
            Answer questions ONLY based on the provided context.
            If the answer is not in the context, say "I don't have that information."
            Always cite the source document.
            
            CONTEXT:
            """ + context;

        // Step 4: Generate answer
        return chatClient.prompt()
            .system(systemPrompt)
            .user(userQuestion)
            .call()
            .content();
    }
}
```

**Thursday — Spring AI QuestionAnswerAdvisor (Built-in RAG)**
```java
// Spring AI has a built-in RAG advisor: QuestionAnswerAdvisor
// This is the recommended way to do RAG in Spring AI

@Configuration
public class RagConfiguration {

    @Bean
    public ChatClient ragChatClient(ChatModel chatModel, VectorStore vectorStore) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                new QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.defaults()
                        .withTopK(5)
                        .withSimilarityThreshold(0.65)
                )
            )
            .defaultSystem("""
                You are a payment policy assistant.
                Answer questions based on the retrieved context.
                Always cite your sources.
                If the context doesn't contain the answer, say so.
                """)
            .build();
    }
}

// Usage is just a regular chat call — the advisor handles retrieval automatically:
@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final ChatClient ragChatClient;

    @PostMapping("/query")
    public RagResponse query(@RequestBody QueryRequest request) {
        String answer = ragChatClient.prompt()
            .user(request.question())
            .call()
            .content();

        return new RagResponse(answer);
    }

    record QueryRequest(String question, String sessionId) {}
    record RagResponse(String answer) {}
}
```

**Friday — Document Ingestion Pipeline (Production Version)**
```java
// Production ingestion: handles multiple file types, deduplication, error recovery

@Service
@Slf4j
public class DocumentIngestionPipeline {

    private final VectorStore vectorStore;
    private final TokenTextSplitter splitter;

    // Supported readers
    private static final Map<String, Function<Resource, DocumentReader>> READER_MAP = Map.of(
        "pdf",  resource -> new PagePdfDocumentReader(resource),
        "txt",  resource -> new TextReader(resource),
        "md",   resource -> new TextReader(resource)
    );

    @Transactional
    public IngestionResult ingest(Resource resource, Map<String, Object> metadata) {
        String filename = resource.getFilename();
        String extension = getExtension(filename);

        DocumentReader reader = READER_MAP.getOrDefault(extension,
            r -> new TextReader(r)).apply(resource);

        List<Document> rawDocs = reader.get();
        log.info("Read {} pages/sections from {}", rawDocs.size(), filename);

        // Enrich metadata
        List<Document> enriched = rawDocs.stream()
            .map(doc -> enrichMetadata(doc, metadata, filename))
            .collect(Collectors.toList());

        // Split into chunks
        List<Document> chunks = splitter.apply(enriched);
        log.info("Split into {} chunks", chunks.size());

        // Store in vector DB (Spring AI embeds automatically)
        vectorStore.add(chunks);

        return new IngestionResult(filename, rawDocs.size(), chunks.size());
    }

    private Document enrichMetadata(Document doc, Map<String, Object> metadata, String filename) {
        Map<String, Object> enriched = new HashMap<>(doc.getMetadata());
        enriched.putAll(metadata);
        enriched.put("filename", filename);
        enriched.put("ingested_at", LocalDateTime.now().toString());
        return new Document(doc.getContent(), enriched);
    }

    record IngestionResult(String filename, int pages, int chunks) {}
}
```

---

## Week 2: Advanced Retrieval

**Monday–Tuesday — Hybrid Search (Keyword + Semantic)**
```java
// Pure semantic search fails for: exact IDs, acronyms, product codes
// "Find transactions with code ERR-429" → semantic search is bad at this
// Hybrid search combines: BM25 keyword search + semantic vector search

// In pgvector with pg_trgm for keyword search:
// SELECT id, content, metadata,
//     (0.5 * (1 - (embedding <=> :vec))) + (0.5 * similarity(content, :query)) AS hybrid_score
// FROM document_embeddings
// WHERE content % :query  -- trigram match (keyword filter)
//    OR embedding <=> :vec < 0.5  -- semantic match
// ORDER BY hybrid_score DESC
// LIMIT 10;

// Reciprocal Rank Fusion (RRF) — merge keyword + semantic result lists:
public List<Document> hybridSearch(String query, float[] queryEmbedding, int topK) {
    // Get keyword results (BM25 ranking)
    List<Document> keywordResults = jdbcTemplate.query(
        "SELECT id, content, metadata FROM document_embeddings " +
        "WHERE content @@ plainto_tsquery(:query) " +
        "ORDER BY ts_rank(to_tsvector(content), plainto_tsquery(:query)) DESC LIMIT 20",
        Map.of("query", query), documentRowMapper
    );

    // Get semantic results
    List<Document> semanticResults = vectorStore.similaritySearch(
        SearchRequest.query(query).withTopK(20)
    );

    // RRF merge: score = sum(1 / (rank + 60)) for each result list
    return reciprocalRankFusion(keywordResults, semanticResults, topK);
}

private List<Document> reciprocalRankFusion(
        List<Document> list1, List<Document> list2, int topK) {
    Map<String, Double> scores = new HashMap<>();
    for (int i = 0; i < list1.size(); i++) {
        scores.merge(list1.get(i).getId(), 1.0 / (i + 60), Double::sum);
    }
    for (int i = 0; i < list2.size(); i++) {
        scores.merge(list2.get(i).getId(), 1.0 / (i + 60), Double::sum);
    }
    // Sort by score, return topK
    return scores.entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .limit(topK)
        .map(e -> findDocById(e.getKey()))
        .collect(Collectors.toList());
}
```

**Wednesday — Query Transformation**
```java
// Problem: user queries are often vague or poorly worded
// Solution: transform the query before retrieval

// 1. Query rewriting: make the query more specific
String rewriteQuery(String originalQuery) {
    return chatClient.prompt()
        .system("""
            Rewrite the user's question to be more specific and searchable.
            The question will be used to search payment policy documents.
            Return ONLY the rewritten question, nothing else.
            """)
        .user(originalQuery)
        .call()
        .content();
}

// 2. HyDE (Hypothetical Document Embedding):
//    Generate a hypothetical answer → embed it → search with that embedding
//    Works because: the ideal answer is closer to relevant chunks than the question
String generateHypotheticalAnswer(String question) {
    return chatClient.prompt()
        .system("""
            Write a brief hypothetical answer to this question, as if you were
            an expert writing a policy document. Be specific and detailed.
            This is used for search, not shown to the user.
            """)
        .user(question)
        .call()
        .content();
}

// Usage with HyDE:
public String queryWithHyDE(String userQuestion) {
    String hypotheticalAnswer = generateHypotheticalAnswer(userQuestion);

    // Embed the hypothetical answer (not the question)
    List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.query(hypotheticalAnswer).withTopK(5)
    );

    return generateAnswer(userQuestion, docs);
}

// 3. Multi-query retrieval: generate multiple query variants → merge results
List<String> generateQueryVariants(String question) {
    String variants = chatClient.prompt()
        .system("Generate 3 different ways to search for the answer to this question. One per line.")
        .user(question)
        .call()
        .content();
    return Arrays.asList(variants.split("\n"));
}
```

**Thursday–Friday — Re-ranking**
```java
// After retrieval: re-rank the top-K results using an LLM or cross-encoder
// Re-ranking is expensive but dramatically improves precision

// Method 1: LLM-based re-ranking (simple, no extra model)
public List<RankedDocument> rerankWithLLM(String query, List<Document> candidates) {
    List<RankedDocument> ranked = new ArrayList<>();

    for (Document doc : candidates) {
        String relevanceScore = chatClient.prompt()
            .system("""
                Rate how relevant this document is to the query.
                Respond with ONLY a number from 0-10.
                10 = perfectly answers the query.
                0 = completely irrelevant.
                """)
            .user("Query: " + query + "\n\nDocument: " + doc.getContent())
            .call()
            .content();

        int score = Integer.parseInt(relevanceScore.trim());
        ranked.add(new RankedDocument(doc, score));
    }

    return ranked.stream()
        .sorted(Comparator.comparingInt(RankedDocument::score).reversed())
        .collect(Collectors.toList());
}

// Method 2: Cohere Rerank API (better quality, external call)
// POST https://api.cohere.ai/v1/rerank
// {
//   "model": "rerank-english-v3.0",
//   "query": "payment refund policy",
//   "documents": ["doc1...", "doc2...", ...],
//   "top_n": 5
// }
// Returns documents ranked by relevance score

record RankedDocument(Document document, int score) {}
```

---

## Week 3: Conversational RAG + Context

**Monday–Tuesday — Conversation History in RAG**
```java
// Conversational RAG: maintain chat history across turns
// Challenge: later questions refer to earlier context
// "What is the refund policy?" → "How long does it take?" (refers to refund)

@Service
public class ConversationalRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final Map<String, List<Message>> sessions = new ConcurrentHashMap<>();

    public String chat(String sessionId, String userMessage) {
        List<Message> history = sessions.getOrDefault(sessionId, new ArrayList<>());

        // Step 1: Reformulate question with conversation context
        String searchQuery = reformulateQuery(userMessage, history);

        // Step 2: Retrieve relevant documents
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(searchQuery).withTopK(5)
        );

        // Step 3: Build prompt with history + retrieved context
        String context = buildContext(docs);

        String answer = chatClient.prompt()
            .system(buildSystemPrompt(context))
            .messages(history)        // include conversation history
            .user(userMessage)
            .call()
            .content();

        // Step 4: Update session history
        history.add(new UserMessage(userMessage));
        history.add(new AssistantMessage(answer));

        // Keep history bounded (prevent context overflow)
        if (history.size() > 20) {
            history = history.subList(history.size() - 20, history.size());
        }
        sessions.put(sessionId, history);

        return answer;
    }

    private String reformulateQuery(String currentMessage, List<Message> history) {
        if (history.isEmpty()) return currentMessage;

        return chatClient.prompt()
            .system("""
                Given the conversation history and the new message,
                reformulate the new message as a standalone search query.
                Return ONLY the reformulated query.
                """)
            .messages(history)
            .user("New message: " + currentMessage)
            .call()
            .content();
    }
}
```

**Wednesday — Source Citation and Transparency**
```java
// Production RAG: always cite sources
// Users need to trust the answers → show where it came from

record RagAnswer(
    String answer,
    List<Citation> sources,
    double confidence
) {}

record Citation(
    String source,
    String section,
    int page,
    String excerpt  // first 100 chars of the relevant chunk
) {}

public RagAnswer queryWithCitations(String question) {
    List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.query(question).withTopK(5)
    );

    String context = docs.stream()
        .map(doc -> String.format("[SOURCE_%d] %s\n%s",
            docs.indexOf(doc) + 1,
            doc.getMetadata().get("source"),
            doc.getContent()))
        .collect(Collectors.joining("\n\n"));

    String answer = chatClient.prompt()
        .system("""
            Answer the question using the provided sources.
            Cite sources as [SOURCE_1], [SOURCE_2], etc.
            Only cite sources that directly support your answer.
            
            SOURCES:
            """ + context)
        .user(question)
        .call()
        .content();

    List<Citation> citations = docs.stream()
        .map(doc -> new Citation(
            (String) doc.getMetadata().get("source"),
            (String) doc.getMetadata().get("section"),
            ((Number) doc.getMetadata().getOrDefault("page", 0)).intValue(),
            doc.getContent().substring(0, Math.min(100, doc.getContent().length()))
        ))
        .collect(Collectors.toList());

    return new RagAnswer(answer, citations, 0.85);
}
```

**Thursday–Friday — RAG Failure Modes**
```
Common RAG failures and how to fix them:

1. RETRIEVAL FAILURE: correct answer exists but wasn't retrieved
   Symptoms: LLM says "I don't have that information" but the doc exists
   Causes:   - Chunk size too large (embedding averages too much meaning)
             - Similarity threshold too high (relevant docs filtered out)
             - Query embedding is too vague
   Fix:      - Reduce chunk size (try 256-400 tokens)
             - Lower threshold (0.5 instead of 0.7)
             - HyDE query transformation
             - Hybrid search

2. HALLUCINATION: LLM makes up information not in the retrieved chunks
   Symptoms: confident-sounding wrong answer
   Causes:   - Prompt doesn't constrain enough ("only use provided context")
             - Retrieved chunks don't contain the answer
             - Temperature too high
   Fix:      - Stronger system prompt ("ONLY answer from context")
             - Temperature=0.0 or 0.1
             - Return "I don't know" when confidence is low

3. CONTEXT POISONING: retrieved irrelevant chunks confuse the LLM
   Symptoms: answer is off-topic or mixes multiple unrelated topics
   Causes:   - topK too high (too many chunks returned)
             - Threshold too low (irrelevant docs included)
   Fix:      - Reduce topK (3-5 is usually right)
             - Raise threshold (0.7+)
             - Re-ranking to filter irrelevant results

4. STALENESS: vector DB has old data, new policies not yet ingested
   Symptoms: correct-sounding answer that's outdated
   Fix:      - Automated re-ingestion pipeline (cron job or event-driven)
             - Add version/updated_at to metadata
             - Filter by date in retrieval
```

---

## Week 4: Production RAG Application

**Build a Payment Policy Q&A System**
```java
// Full production RAG: ingest PDFs, answer questions, cite sources, trace calls

@RestController
@RequestMapping("/api/policy")
public class PolicyQAController {

    private final ConversationalRagService ragService;
    private final DocumentIngestionPipeline ingestionPipeline;

    // Ingest a new policy document
    @PostMapping("/ingest")
    @PreAuthorize("hasRole('ADMIN')")
    public IngestionResult ingest(
            @RequestParam MultipartFile file,
            @RequestParam String documentType,
            @RequestParam String department) throws IOException {

        Resource resource = new ByteArrayResource(file.getBytes());
        return ingestionPipeline.ingest(resource, Map.of(
            "document_type", documentType,
            "department", department,
            "uploaded_by", getCurrentUser()
        ));
    }

    // Query with conversation history
    @PostMapping("/ask")
    public RagAnswer ask(
            @RequestBody AskRequest request,
            @AuthenticationPrincipal UserDetails user) {

        return ragService.queryWithCitations(request.question());
    }

    // Chat with session history
    @PostMapping("/chat")
    public ChatResponse chat(
            @RequestBody ChatRequest request,
            HttpSession session) {

        String sessionId = session.getId();
        String answer = ragService.chat(sessionId, request.message());
        return new ChatResponse(answer, sessionId);
    }

    record AskRequest(String question) {}
    record ChatRequest(String message) {}
    record ChatResponse(String answer, String sessionId) {}
}
```

```java
// Observability: trace every RAG call
// Use Spring AOP to log retrieval quality

@Aspect
@Component
@Slf4j
public class RagObservabilityAspect {

    @Around("@annotation(TraceRag)")
    public Object traceRagCall(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;

        log.info("RAG call: method={}, duration={}ms",
            pjp.getSignature().getName(), duration);

        return result;
    }
}

// Metrics to track in production:
// - retrieval_latency_ms: how long similarity search takes
// - chunks_retrieved: how many chunks returned per query
// - answer_latency_ms: how long LLM takes to generate
// - queries_without_results: % of queries where nothing was retrieved
// - user_feedback_score: thumbs up/down from users (build this into the UI)
```
