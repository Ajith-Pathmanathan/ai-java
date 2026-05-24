# Module 04: MCP + AI Agents
**Duration:** 3 weeks (2 hours/day)
**Goal:** Understand MCP (Model Context Protocol), build MCP servers in Java, and create autonomous AI agents that take real actions.

---

## The Big Idea

```
RAG: LLM reads documents and answers questions.
     LLM is PASSIVE — it only generates text.

Agents: LLM can TAKE ACTIONS:
     - Query a database
     - Call an API
     - Read/write files
     - Execute code
     - Make decisions about what to do next

MCP (Model Context Protocol):
     Anthropic's standard for connecting LLMs to tools.
     Your Spring Boot app → MCP server → LLM uses your tools
     
     Think of MCP as USB for AI tools.
     Just as USB standardizes how devices connect to computers,
     MCP standardizes how AI models connect to tools and data sources.

Without MCP: every AI system builds its own custom tool integration.
With MCP:    one standard, every LLM can use every tool.
```

---

## Week 1: MCP Deep Dive

**Monday — What is MCP?**
```
MCP = Model Context Protocol (Anthropic, 2024)

MCP defines:
  Tools:     functions the LLM can call (e.g., queryPaymentDatabase)
  Resources: data the LLM can read (e.g., policy documents, schemas)
  Prompts:   pre-built prompt templates the LLM can use

MCP vs Function Calling:
  Function calling (OpenAI): proprietary, only for OpenAI models
  MCP:                       open standard, any LLM that supports it

Architecture:
  Claude Desktop / Cursor / your app  ← MCP Client
        ↕ (JSON-RPC over stdio/HTTP)
  Your Spring Boot MCP Server         ← MCP Server
        ↕
  PostgreSQL, REST APIs, File System  ← Your actual systems

MCP servers expose tools → Claude calls them → results come back → Claude continues reasoning.
This is how Claude Code works: it runs MCP servers that give it file system access.
```

**Tuesday–Wednesday — Building a Spring Boot MCP Server**
```java
// Spring AI MCP Server dependency:
// <dependency>
//   <groupId>org.springframework.ai</groupId>
//   <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
// </dependency>

// Define MCP tools as Spring beans:
@Service
public class PaymentMcpTools {

    private final PaymentRepository paymentRepository;

    // Tool: query payment by ID
    @Tool(name = "get_payment",
          description = "Retrieve payment details by payment ID. Returns payment status, amount, currency, merchant, and failure reason if applicable.")
    public PaymentDetails getPayment(
            @ToolParam(name = "payment_id", description = "The payment ID, e.g. PAY-12345")
            String paymentId) {

        return paymentRepository.findById(paymentId)
            .map(payment -> new PaymentDetails(
                payment.getId(),
                payment.getStatus(),
                payment.getAmount(),
                payment.getCurrency(),
                payment.getMerchantName(),
                payment.getFailureReason()
            ))
            .orElseThrow(() -> new ToolException("Payment " + paymentId + " not found"));
    }

    // Tool: search payments by customer
    @Tool(name = "search_payments",
          description = "Search payments for a customer. Returns list of recent payments with status and amounts.")
    public List<PaymentSummary> searchPayments(
            @ToolParam(name = "customer_id", description = "Customer ID to search payments for")
            String customerId,
            @ToolParam(name = "limit", description = "Maximum number of results (1-50)", required = false)
            Integer limit) {

        int maxResults = limit != null ? Math.min(limit, 50) : 10;
        return paymentRepository.findByCustomerId(customerId, PageRequest.of(0, maxResults))
            .stream()
            .map(p -> new PaymentSummary(p.getId(), p.getStatus(), p.getAmount(), p.getCreatedAt()))
            .collect(Collectors.toList());
    }

    // Tool: get refund eligibility
    @Tool(name = "check_refund_eligibility",
          description = "Check if a payment is eligible for refund. Returns eligibility status and reason.")
    public RefundEligibility checkRefundEligibility(
            @ToolParam(name = "payment_id") String paymentId) {

        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow(() -> new ToolException("Payment not found"));

        boolean eligible = payment.getStatus() == PaymentStatus.COMPLETED
            && payment.getCreatedAt().isAfter(LocalDateTime.now().minusDays(30));

        String reason = eligible ? "Payment is within 30-day refund window"
                                 : "Payment is outside 30-day refund window";

        return new RefundEligibility(paymentId, eligible, reason);
    }
}

// Register tools with Spring AI MCP:
@Configuration
public class McpConfiguration {

    @Bean
    public ToolCallbackProvider paymentTools(PaymentMcpTools paymentMcpTools) {
        return MethodToolCallbackProvider.builder()
            .toolObjects(paymentMcpTools)
            .build();
    }
}
```

```yaml
# application.yml for MCP server:
spring:
  ai:
    mcp:
      server:
        name: payment-mcp-server
        version: 1.0.0
        transport: STDIO   # or HTTP for remote MCP
        tools-change-notification: true
```

**Thursday — MCP Resources and Prompts**
```java
// MCP Resources: expose data that the LLM can read

@Component
public class PaymentSchemaResource implements McpResourceProvider {

    @Override
    public List<McpSchema.Resource> listResources() {
        return List.of(
            new McpSchema.Resource(
                "payment://schema",
                "Payment Database Schema",
                "The database schema for payment tables — useful for generating SQL queries",
                "text/plain",
                null
            )
        );
    }

    @Override
    public McpSchema.ReadResourceResult readResource(McpSchema.ReadResourceRequest request) {
        if ("payment://schema".equals(request.uri())) {
            String schema = """
                Table: payments
                  id          UUID PRIMARY KEY
                  customer_id UUID NOT NULL
                  amount      DECIMAL(10,2)
                  currency    VARCHAR(3)
                  status      VARCHAR(20)  -- PENDING, COMPLETED, FAILED, REFUNDED
                  merchant_id UUID
                  failure_reason TEXT
                  created_at  TIMESTAMPTZ
                
                Table: refunds
                  id          UUID PRIMARY KEY
                  payment_id  UUID REFERENCES payments(id)
                  amount      DECIMAL(10,2)
                  reason      TEXT
                  created_at  TIMESTAMPTZ
                """;
            return new McpSchema.ReadResourceResult(
                List.of(new McpSchema.TextContent(schema))
            );
        }
        throw new ToolException("Resource not found: " + request.uri());
    }
}

// MCP Prompts: pre-built prompt templates for common workflows
@Component
public class PaymentPrompts implements McpPromptProvider {

    @Override
    public List<McpSchema.Prompt> listPrompts() {
        return List.of(
            new McpSchema.Prompt(
                "payment_dispute_analysis",
                "Analyse a payment dispute and recommend resolution",
                List.of(
                    new McpSchema.PromptArgument("payment_id", "The disputed payment ID", true),
                    new McpSchema.PromptArgument("customer_complaint", "The customer's complaint text", true)
                )
            )
        );
    }
}
```

**Friday — MCP HTTP Transport (for Remote Servers)**
```yaml
# HTTP transport: MCP server as a REST endpoint
# Use when: your MCP server is remote, or needs to be shared across multiple clients

spring:
  ai:
    mcp:
      server:
        transport: HTTP
        port: 8080
        path: /mcp

# MCP client configuration (your main app calling a remote MCP server):
spring:
  ai:
    mcp:
      client:
        servers:
          payment-server:
            transport: HTTP
            url: http://payment-mcp-server:8080/mcp
```

```java
// MCP client: connect to a remote MCP server and use its tools
@Service
public class McpClientService {

    private final McpClient mcpClient;

    public List<McpSchema.Tool> listAvailableTools() {
        return mcpClient.listTools().tools();
    }

    // Call a tool on the remote MCP server:
    public String callPaymentTool(String paymentId) {
        McpSchema.CallToolResult result = mcpClient.callTool(
            new McpSchema.CallToolRequest(
                "get_payment",
                Map.of("payment_id", paymentId)
            )
        );
        return result.content().get(0).text();
    }
}
```

---

## Week 2: AI Agents with Spring AI

**Monday–Tuesday — What is an Agent?**
```
Agent = LLM that can:
  1. REASON about what to do (given a goal)
  2. ACT by calling tools
  3. OBSERVE the results
  4. LOOP: reason about next step based on results

ReAct Pattern (Reasoning + Acting):
  Thought:  "I need to find the payment details for PAY-123"
  Action:   get_payment(payment_id="PAY-123")
  Observation: {status: "FAILED", reason: "Insufficient funds"}
  Thought:  "The payment failed due to insufficient funds. Let me check if refund is applicable."
  Action:   check_refund_eligibility(payment_id="PAY-123")
  Observation: {eligible: false, reason: "Payment failed, not eligible for refund"}
  Thought:  "I now have enough information to answer the customer."
  Answer:   "Your payment PAY-123 failed due to insufficient funds..."

This loop continues until the agent reaches an answer or a stop condition.
```

**Wednesday–Thursday — Building an Agent with Spring AI**
```java
// Spring AI Agent: a chat client with tools attached

@Service
public class PaymentSupportAgent {

    private final ChatClient agentChatClient;

    @Autowired
    public PaymentSupportAgent(
            ChatModel chatModel,
            PaymentMcpTools paymentTools) {

        // The agent: ChatClient + tools
        this.agentChatClient = ChatClient.builder(chatModel)
            .defaultTools(paymentTools)  // agent can call these tools
            .defaultSystem("""
                You are a payment support specialist with access to the payment database.
                
                When a customer asks about their payment:
                1. Use the get_payment tool to retrieve payment details.
                2. Use check_refund_eligibility if they ask about refunds.
                3. Use search_payments to find recent transactions if needed.
                
                Always verify payment details before making any claims.
                Never guess — use the tools to get accurate information.
                
                SECURITY: Only access payments belonging to the authenticated customer.
                """)
            .build();
    }

    public String handleCustomerQuery(String customerId, String question) {
        // The LLM will automatically call tools as needed
        // Spring AI handles the ReAct loop internally
        return agentChatClient.prompt()
            .system("AUTHENTICATED_CUSTOMER_ID: " + customerId)
            .user(question)
            .call()
            .content();
    }
}
```

**Friday — Agent Security and Guardrails**
```java
// Agents are powerful and dangerous without guardrails
// An agent can call tools repeatedly — cost and security risks

// 1. Tool call limits: prevent runaway agents
@Configuration
public class AgentConfiguration {

    @Bean
    public ChatClient guardedAgentClient(ChatModel chatModel, PaymentMcpTools tools) {
        return ChatClient.builder(chatModel)
            .defaultTools(tools)
            .defaultAdvisors(
                new MaxTokensAdvisor(4000),  // limit total tokens per conversation
                new ToolCallCountAdvisor(10) // max 10 tool calls per request
            )
            .build();
    }
}

// 2. Tool-level authorization: ensure tools only access authorized data
@Tool(name = "get_payment")
public PaymentDetails getPayment(String paymentId, @AuthenticatedCustomer String customerId) {
    Payment payment = paymentRepository.findById(paymentId)
        .orElseThrow(() -> new ToolException("Payment not found"));

    // SECURITY CHECK: payment must belong to the authenticated customer
    if (!payment.getCustomerId().equals(customerId)) {
        throw new ToolException("Access denied: payment does not belong to this customer");
    }

    return mapToDetails(payment);
}

// 3. Sensitive action confirmation: require explicit approval for destructive actions
@Tool(name = "initiate_refund",
      description = "Initiate a refund for a payment. REQUIRES explicit customer confirmation.")
public RefundResult initiateRefund(String paymentId, boolean customerConfirmed) {
    if (!customerConfirmed) {
        throw new ToolException("Refund not initiated: customer confirmation required. " +
            "Please confirm: 'Yes, initiate a refund for payment " + paymentId + "'");
    }
    // ... process refund
}

// 4. Audit trail: log every tool call
@Aspect
@Component
@Slf4j
public class AgentAuditAspect {

    @Around("@annotation(Tool)")
    public Object auditToolCall(ProceedingJoinPoint pjp) throws Throwable {
        String toolName = pjp.getSignature().getName();
        Object[] args = pjp.getArgs();
        log.info("AGENT_TOOL_CALL: tool={}, args={}", toolName, Arrays.toString(args));

        Object result = pjp.proceed();
        log.info("AGENT_TOOL_RESULT: tool={}, success=true", toolName);
        return result;
    }
}
```

---

## Week 3: Multi-Agent Systems

**Monday–Tuesday — Agent Orchestration Patterns**
```
Multi-agent: multiple specialized agents working together

Pattern 1: Sequential (Pipeline)
  Agent A (search) → Agent B (analyze) → Agent C (write report)
  Each agent's output is the next agent's input

Pattern 2: Parallel
  Orchestrator → Agent A (search web)
             → Agent B (search database)
             → Agent C (search documents)
  All run simultaneously → Orchestrator merges results

Pattern 3: Hierarchical
  Manager Agent → delegates subtasks to Specialist Agents
                → collects results → synthesizes final answer

Payment support example:
  Customer Query
       ↓
  Router Agent (classify: technical, billing, fraud?)
       ↓
  Specialist Agent (technical / billing / fraud)
       ↓
  Response Agent (format + quality check)
       ↓
  Response to customer
```

**Wednesday–Thursday — Building a Multi-Agent Pipeline**
```java
// Multi-agent payment support system

@Service
public class PaymentSupportOrchestrator {

    private final ChatClient routerAgent;
    private final ChatClient billingAgent;
    private final ChatClient technicalAgent;
    private final ChatClient fraudAgent;
    private final ChatClient responseAgent;

    public String handleQuery(String customerId, String query) {
        // Step 1: Router agent classifies the query
        String category = routerAgent.prompt()
            .system("Classify the query as: BILLING, TECHNICAL, or FRAUD. Return ONLY the category.")
            .user(query)
            .call()
            .content()
            .trim();

        // Step 2: Specialist agent handles it
        String specialistAnalysis = switch (category) {
            case "BILLING"    -> billingAgent.prompt()
                .system("You are a billing specialist. Customer ID: " + customerId)
                .user(query).call().content();
            case "TECHNICAL"  -> technicalAgent.prompt()
                .system("You are a technical support specialist. Customer ID: " + customerId)
                .user(query).call().content();
            case "FRAUD"      -> fraudAgent.prompt()
                .system("You are a fraud investigation specialist. Customer ID: " + customerId)
                .user(query).call().content();
            default -> "Unable to classify query. Please contact support directly.";
        };

        // Step 3: Response agent formats the final reply
        return responseAgent.prompt()
            .system("""
                Format this technical analysis as a friendly customer-facing response.
                Be empathetic, clear, and concise.
                Include next steps if applicable.
                """)
            .user("Customer query: " + query + "\n\nAnalysis: " + specialistAnalysis)
            .call()
            .content();
    }
}

// Parallel agent execution with CompletableFuture:
public CompletableFuture<String> parallelResearch(String query) {
    CompletableFuture<String> webResults = CompletableFuture.supplyAsync(
        () -> webSearchAgent.prompt().user(query).call().content()
    );
    CompletableFuture<String> dbResults = CompletableFuture.supplyAsync(
        () -> databaseAgent.prompt().user(query).call().content()
    );
    CompletableFuture<String> docResults = CompletableFuture.supplyAsync(
        () -> documentAgent.prompt().user(query).call().content()
    );

    return CompletableFuture.allOf(webResults, dbResults, docResults)
        .thenApply(v -> synthesizerAgent.prompt()
            .system("Synthesize these research results into a comprehensive answer:")
            .user("Web: " + webResults.join() +
                  "\nDatabase: " + dbResults.join() +
                  "\nDocuments: " + docResults.join())
            .call()
            .content()
        );
}
```

**Friday — Agent Failure Modes and Recovery**
```
Common agent failure modes:

1. INFINITE LOOP: agent keeps calling tools without reaching a conclusion
   Fix: set max_iterations limit (10 tool calls max), timeout per request

2. TOOL HALLUCINATION: agent calls a tool with made-up parameters
   Fix: validate all tool inputs, return clear error messages from tools

3. CASCADING FAILURE: one tool fails → agent panics → wrong answer
   Fix: tools should return structured errors, agent handles them gracefully

4. COST EXPLOSION: agent makes 100 tool calls for a simple question
   Fix: tool call budget, log and alert when budget exceeded

5. WRONG TOOL SELECTION: agent uses the wrong tool for the task
   Fix: clear tool descriptions (most important!), few-shot examples in system prompt

Best practices for production agents:
  ✅ Every tool has a clear, specific description
  ✅ Max tool calls per request (10 is a good default)
  ✅ Request timeout (30 seconds for customer-facing)
  ✅ Every tool call is logged with inputs + outputs
  ✅ Fallback: if agent fails, route to human
  ✅ Test agents with adversarial inputs (try to make them loop or hallucinate)
```
