# Module 05: LangChain4j + Agentic Workflows — Progress Tracker

**Start Date:** _______________ | **Target:** 3 weeks | **Actual Completion:** _______________

---

## Week 1: LangChain4j Fundamentals

| Task | Done |
|------|------|
| Add LangChain4j OpenAI starter to pom.xml — configure API key | ☐ |
| Create `PaymentAssistant` interface with `@AiService` and `@SystemMessage` | ☐ |
| Use `@UserMessage` annotation — compare cleanliness to Spring AI ChatClient | ☐ |
| Implement structured output: `PaymentExtraction` record via `@AiService` | ☐ |
| Implement `PaymentTools` class with `@Tool` on 3 methods | ☐ |
| Wire tools to AI Service via `AiServices.builder().tools()` | ☐ |
| Implement streaming with `TokenStream` → `Flux<String>` SSE endpoint | ☐ |
| Research Q1 answered | ☐ |

---

## Week 2: Memory-Enabled Agents

| Task | Done |
|------|------|
| Implement `PostgresChatMemoryStore` with JDBC (getMessages, updateMessages, deleteMessages) | ☐ |
| Configure AI Service with `chatMemoryProvider` using PostgreSQL store | ☐ |
| Test memory persistence: send 5 messages → restart server → ask "What did I say first?" | ☐ |
| Implement `MessageWindowChatMemory` (20 messages) vs `TokenWindowChatMemory` (8000 tokens) | ☐ |
| Implement basic summarizing memory: when > 20 messages, summarize oldest 10 | ☐ |
| Implement LangChain4j RAG: `EmbeddingStoreContentRetriever` + `@AiService` | ☐ |
| Combine RAG + memory in one AI Service — test that it cites sources AND remembers context | ☐ |
| Research Q2, Q3 answered | ☐ |

---

## Week 3: Complex Agentic Workflows

| Task | Done |
|------|------|
| Build `PaymentDisputeWorkflow`: 3-step chain (extract → assess → respond) | ☐ |
| Implement explicit service chain (3 AI Services orchestrated in Java) | ☐ |
| Implement same workflow as a single agent with tools — compare results | ☐ |
| Write unit tests: mock `ChatLanguageModel` for AI Service tests | ☐ |
| Test tool failure recovery: tool throws exception → agent tries different approach | ☐ |
| Compare Spring AI vs LangChain4j: build same feature in both — document which was easier | ☐ |
| Write a 1-page decision guide: when to use Spring AI vs LangChain4j | ☐ |
| Research Q4, Q5, Q6, Q7, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build a memory-enabled agent with LangChain4j that remembers conversations across requests and calls external tools? Can you explain when to use LangChain4j vs Spring AI? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| LangChain4j @AiService pattern | |
| Structured output with LangChain4j | |
| Tool definition with @Tool annotation | |
| Persistent ChatMemoryStore (PostgreSQL) | |
| Memory strategies (window, token, summarizing) | |
| LangChain4j RAG pipeline | |
| Multi-step agent chaining | |
| Streaming with TokenStream | |
| Spring AI vs LangChain4j trade-offs | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Fundamentals | | |
| Week 2 — Memory Agents | | |
| Week 3 — Complex Workflows | | |
| **Total** | | |
