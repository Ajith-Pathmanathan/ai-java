# Module 04: MCP + AI Agents — Progress Tracker

**Start Date:** _______________ | **Target:** 3 weeks | **Actual Completion:** _______________

---

## Week 1: MCP Deep Dive

| Task | Done |
|------|------|
| Read the official MCP spec at modelcontextprotocol.io — draw the architecture diagram | ☐ |
| Add `spring-ai-mcp-server-spring-boot-starter` to a new Spring Boot project | ☐ |
| Implement `get_payment` tool with `@Tool` annotation | ☐ |
| Implement `search_payments` tool with customer_id filter | ☐ |
| Implement `check_refund_eligibility` tool | ☐ |
| Configure MCP server in application.yml (stdio or HTTP transport) | ☐ |
| Test with Claude Desktop (add your MCP server to claude_desktop_config.json) | ☐ |
| Implement an MCP Resource for the payment schema | ☐ |
| Research Q1, Q2 answered | ☐ |

---

## Week 2: AI Agents

| Task | Done |
|------|------|
| Build a basic agent: ChatClient with tools attached | ☐ |
| Ask the agent: "Why was payment PAY-123 declined?" — observe it call tools automatically | ☐ |
| Trace the ReAct loop: add logging to every tool call to see the agent's reasoning | ☐ |
| Add max tool call limit to prevent runaway agents | ☐ |
| Test security: ask agent about a payment belonging to another customer — verify it refuses | ☐ |
| Add audit logging: every tool call logged with inputs, outputs, and customer ID | ☐ |
| Test adversarial input: try a prompt injection via the query field | ☐ |
| Research Q3, Q4, Q5 answered | ☐ |

---

## Week 3: Multi-Agent Systems

| Task | Done |
|------|------|
| Build a router agent: classify queries as BILLING, TECHNICAL, or FRAUD | ☐ |
| Build 3 specialist agents: each with domain-specific system prompt and tools | ☐ |
| Wire them together in PaymentSupportOrchestrator | ☐ |
| Test sequential pipeline: router → specialist → response formatter | ☐ |
| Test parallel execution with CompletableFuture.allOf — measure latency improvement | ☐ |
| Simulate tool failure: mock one tool to throw an exception — verify graceful degradation | ☐ |
| Write integration test: full agent flow with mocked tools — verify correct tool calls | ☐ |
| Research Q6, Q7, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you build an MCP server in Spring Boot that Claude Desktop can connect to? Can you build an agent that automatically calls tools to answer payment questions? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| MCP protocol understanding (tools, resources, prompts) | |
| Building MCP tools with `@Tool` annotation | |
| MCP stdio vs HTTP transport | |
| Agent architecture with Spring AI | |
| ReAct loop understanding | |
| Agent security (authorization, audit, rate limiting) | |
| Multi-agent orchestration (sequential + parallel) | |
| Tool description quality | |
| Agent testing strategies | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — MCP Server | | |
| Week 2 — Agents | | |
| Week 3 — Multi-Agent | | |
| **Total** | | |
