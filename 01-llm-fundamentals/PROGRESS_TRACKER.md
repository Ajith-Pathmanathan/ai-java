# Module 01: LLM Fundamentals — Progress Tracker

**Start Date:** _______________ | **Target:** 3 weeks | **Actual Completion:** _______________

---

## Week 1: How LLMs Work

| Task | Done |
|------|------|
| Set up Ollama locally: `ollama pull llama3.1` — run a test query | ☐ |
| Create a Spring Boot project with `spring-ai-ollama-spring-boot-starter` | ☐ |
| Send your first chat message via Spring AI `ChatClient` | ☐ |
| Experiment with temperature: send same prompt at 0.0, 0.5, 0.9 — observe differences | ☐ |
| Count tokens for 5 different text sizes using jtokkit | ☐ |
| Implement streaming response with Spring WebFlux SSE endpoint | ☐ |
| Implement structured output: extract `PaymentIntent` from a customer message | ☐ |
| Research Q1, Q2, Q7, Q8 answered | ☐ |

---

## Week 2: Prompt Engineering

| Task | Done |
|------|------|
| Write a system prompt with all 6 parts (persona, context, instructions, constraints, examples, format) | ☐ |
| Implement few-shot classification: 4 categories, 3 examples each | ☐ |
| Implement Chain of Thought prompt for a payment risk decision | ☐ |
| Implement prompt injection guard (pattern detection) | ☐ |
| Write 5 test cases that verify your prompt gives correct output | ☐ |
| Run all 5 test cases 3 times — verify output is consistent (temperature=0.0) | ☐ |
| Research Q3, Q4, Q5, Q6 answered | ☐ |

---

## Week 3: Production Application

| Task | Done |
|------|------|
| Build PaymentSupportController with: input validation + intent extraction + response | ☐ |
| Implement `PromptTemplate` entity and `PromptService` | ☐ |
| Write unit tests for prompt outputs (classification accuracy > 90%) | ☐ |
| Add output sanitisation (redact card numbers from AI output) | ☐ |
| Add token counting to every LLM call — log it for monitoring | ☐ |
| Switch from Ollama to OpenAI (or vice versa) — verify same results | ☐ |
| Research Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build a Spring Boot payment support chatbot with: structured output, few-shot classification, prompt injection guard, and streaming responses? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| Token counting and context window math | |
| Temperature / top-p tuning | |
| System prompt design (6 parts) | |
| Few-shot prompting | |
| Chain of Thought prompting | |
| Structured output (Spring AI entities) | |
| Streaming with Spring WebFlux/SSE | |
| Prompt injection detection | |
| Prompt versioning and testing | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — LLM Mechanics | | |
| Week 2 — Prompt Engineering | | |
| Week 3 — Production App | | |
| **Total** | | |
