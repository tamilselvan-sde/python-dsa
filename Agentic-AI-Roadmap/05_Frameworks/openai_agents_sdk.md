# OpenAI Agents SDK

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

The OpenAI Agents SDK is an open-source Python framework released by OpenAI in March 2025 for building agentic AI applications. It provides a first-party, lightweight SDK for creating agents powered by OpenAI models (GPT-4o, GPT-4o-mini, o1, o3) with built-in tool use, handoffs between agents, guardrails, and tracing. The SDK represents OpenAI's official entry into the agent framework space — the same technology powering OpenAI's internal agent infrastructure.

## 2. Why do we need it?

Existing agent frameworks (LangChain, CrewAI, AutoGen) are third-party abstractions over OpenAI's API. They add complexity, break when OpenAI changes its API, and cannot leverage OpenAI's latest features (response headers, structured outputs, streaming natively). The OpenAI Agents SDK solves:
- **First-party integration:** No abstraction layer — direct access to OpenAI's latest API features
- **Native tool calling:** Structured tool schemas with automatic parsing and validation
- **Agent handoffs:** Seamless delegation between agents (like router agents)
- **Built-in guardrails:** Input and output validation at the framework level
- **Tracing:** OpenTelemetry-based observability with OpenAI's trace visualiser

## 3. Real-world Example

**Customer support routing system:** An incoming support ticket is handled by a triage agent that classifies the issue. It hands off to the appropriate specialist agent (billing, technical, or account). The specialist agent uses tools to resolve the issue (look up account, refund transaction, escalate to human). If confidence is low, the triage agent asks clarifying questions before handing off. All interactions are traced end-to-end.

## 4. Architecture Diagram (ASCII)

```
                 +---------------------------+
                 |    OpenAI Agents SDK      |
                 +---------------------------+
                 |                           |
    +------------+     +--------------------+-----------------+
    |  Agent     |     |  Handoffs          |  Guardrails     |
    |  (LLM +    |     |  (Agent routing)   |  (input/output)|
    |   tools)   |     +--------------------+-----------------+
    +-----+------+
          |
    +-----v------+
    |  Tools     |
    |  (function |
    |   calls)   |
    +------------+
```

## 5. Internal Working

The SDK is built around three core primitives:

1. **Agents:** Each agent has `instructions` (system prompt), `model` (OpenAI model), `tools` (callable functions with schemas), and `handoffs` (list of other agents it can delegate to). Agents process incoming messages and generate responses — either text, tool calls, or handoffs.
2. **Handoffs:** An agent can transfer to another agent. The transfer is communicated via a structured tool call (`transfer_to_<agent_name>`). The receiving agent gets the conversation history.
3. **Guardrails:** Functions that validate input (before the agent processes) or output (after the agent responds). If a guardrail fails, the agent can be instructed to modify its response.

Execution flow: `Runner.run(agent, input)` → agent processes → may call tools → may handoff → returns `RunResult` with final output.

## 6. Production Flow

```
Input → Input Guardrail (validate/transform) →
Agent Selection (triage router) →
Agent Processes (reasoning + tool calls) →
Tool Execution → Output Guardrail (validate) →
Handoff? → Yes → Next Agent → Loop
No → Return Response
```

## 7. HLD

```
                   +----------------------+
                   |   API Gateway        |
                   +----------+-----------+
                              |
                   +----------+-----------+
                   |  OpenAI Agents       |
                   |  SDK Service         |
                   +----------+-----------+
                              |
              +---------------+---------------+
              |               |               |
    +---------v----+   +-----v------+  +-----v------+
    | OpenAI API   |   | Tool       |  | Tracing    |
    | (GPT-4o)     |   | Executor   |  | Backend    |
    +--------------+   | (sandbox)  |  | (OTel)     |
                       +------------+  +------------+
```

## 8. LLD

```
Agent
  ├── name: str
  ├── instructions: str | list[dict]  # system prompt
  ├── model: str | OpenAIChatCompletionsModel
  ├── tools: list[FunctionTool | Toolset]
  ├── handoffs: list[Agent]
  ├── input_guardrails: list[GuardrailFunction]
  ├── output_guardrails: list[GuardrailFunction]
  └── model_settings: ModelSettings

Runner
  ├── run(agent, input, context) -> RunResult
  ├── run_streamed(agent, input, context) -> RunResultStreaming
  └── run_until_handoff(agent, input) -> HandoffRunResult

RunResult
  ├── final_output: Any
  ├── last_agent: Agent
  ├── messages: list[dict]
  └── trace_id: str

FunctionTool
  ├── name: str
  ├── description: str
  ├── params_json_schema: dict
  └── on_invoke_tool(input) -> Any

Handoff
  ├── agent: Agent
  ├── tool_name: str  # auto-generated
  └── on_handoff() -> dict  # optional context

GuardrailFunction
  ├── check_input(ctx, agent, input) -> GuardrailResult
  └── check_output(ctx, agent, output) -> GuardrailResult

Trace
  ├── trace_id: str
  ├── spans: list[Span]  # agent, tool, handoff, guardrail
  └── export() -> OTel format
```

## 9. Python Implementation

```python
from openai.agents import Agent, Runner, FunctionTool, Guardrail, trace
from openai.agents.guardrails import InputGuardrail, OutputGuardrail
from pydantic import BaseModel
import asyncio

# Define a tool with Pydantic schema
class RefundInput(BaseModel):
    order_id: str
    reason: str

async def process_refund(order_id: str, reason: str) -> str:
    """Process a refund for an order."""
    # Call payment API
    return f"Refund for order {order_id} processed. Reason: {reason}"

refund_tool = FunctionTool(
    name="process_refund",
    description="Process a refund for a customer order",
    params_json_schema=RefundInput.model_json_schema(),
    on_invoke_tool=process_refund,
)

class LookupInput(BaseModel):
    customer_id: str

async def lookup_customer(customer_id: str) -> dict:
    """Look up customer account details."""
    return {"name": "Alice", "tier": "gold", "orders": ["ORD-123"]}

lookup_tool = FunctionTool(
    name="lookup_customer",
    description="Look up customer by ID",
    params_json_schema=LookupInput.model_json_schema(),
    on_invoke_tool=lookup_customer,
)

# Define guardrails
async def positive_tone_guardrail(ctx, agent, input_text: str):
    """Ensure input is not abusive."""
    if "angry" in input_text.lower() and "refund" in input_text.lower():
        return Guardrail.fail("Route to senior support with context: customer is frustrated")
    return Guardrail.pass_()

async def pii_guardrail(ctx, agent, output: str):
    """Ensure output contains no PII."""
    if "@" in output:  # simple email check
        return Guardrail.fail("Response should not contain email addresses")
    return Guardrail.pass_()

# Create specialist agents
billing_agent = Agent(
    name="Billing Agent",
    instructions="You handle billing issues. Use refund tool when appropriate.",
    tools=[refund_tool, lookup_tool],
    model="gpt-4o",
    output_guardrails=[OutputGuardrail(pii_guardrail)],
)

tech_support_agent = Agent(
    name="Tech Support Agent",
    instructions="You handle technical issues. Troubleshoot step by step.",
    model="gpt-4o-mini",
)

# Create triage agent with handoffs
triage_agent = Agent(
    name="Triage Agent",
    instructions=(
        "You are a customer support triage agent. "
        "Classify the issue and hand off to the correct specialist. "
        "Ask clarifying questions if uncertain."
    ),
    handoffs=[billing_agent, tech_support_agent],
    input_guardrails=[InputGuardrail(positive_tone_guardrail)],
    model="gpt-4o",
)

# Run with tracing
async def main():
    with trace("customer_support_session"):
        result = await Runner.run(
            triage_agent,
            "I need a refund for order ORD-123. The product arrived damaged.",
            context={"customer_id": "CUST-456"}
        )

        print(f"Final agent: {result.last_agent.name}")
        print(f"Response: {result.final_output}")
        print(f"Trace ID: {result.trace_id}")

        # Check if handoff occurred
        for message in result.messages:
            if message.get("role") == "handoff":
                print(f"Handoff from {message['from_agent']} to {message['to_agent']}")

asyncio.run(main())
```

### Streaming Example

```python
stream = Runner.run_streamed(triage_agent, "My internet is down")
async for event in stream:
    if event.type == "agent_response_chunk":
        print(event.delta, end="", flush=True)
    elif event.type == "tool_call":
        print(f"\n[Tool: {event.tool_name}({event.arguments})]")
    elif event.type == "handoff":
        print(f"\n[Handoff to {event.to_agent.name}]")
```

## 10. Folder Structure

```
openai-agents-app/
├── agents/
│   ├── __init__.py
│   ├── triage_agent.py
│   ├── billing_agent.py
│   └── tech_support_agent.py
├── tools/
│   ├── __init__.py
│   ├── payment_tools.py
│   ├── customer_tools.py
│   └── inventory_tools.py
├── guardrails/
│   ├── __init__.py
│   ├── tone_guardrail.py
│   └── pii_guardrail.py
├── api/
│   └── main.py
├── tests/
├── .env
├── pyproject.toml
└── Dockerfile
```

## 11. Configuration

```python
# config.py
from pydantic_settings import BaseSettings

class OpenAIAgentsConfig(BaseSettings):
    openai_api_key: str = ""
    default_model: str = "gpt-4o"
    tracing_enabled: bool = True
    tracing_endpoint: str = "http://localhost:4318/v1/traces"
    trace_export_schedule_ms: int = 5000

    class Config:
        env_file = ".env"
```

## 12. Flowchart

```
Input
  |
  v
Input Guardrails (validate, reject, or transform)
  |
  v
Triage Agent processes
  |
  +-- Needs handoff? --> Handoff to Specialist Agent
  |                           |
  |                           v
  |                    Specialist processes
  |                           |
  |                    +-- Tool call? --> Execute tool --> Loop
  |                    +-- Respond
  |
  +-- Responds directly
  |
  v
Output Guardrails (validate response)
  |
  v
Return final output
```

## 13. Sequence Diagram

```
User       Runner    TriageAgent  Specialist   Tool     OpenAI API
 |          |           |            |          |          |
 |--run()-->|           |            |          |          |
 |          |--process->|            |          |          |
 |          |           |--LLM call->|          |          |
 |          |           |<--response-|          |          |
 |          |           |            |          |          |
 |          |           + handoff    |          |          |
 |          |           |--transfer->|          |          |
 |          |           |            |--LLM--->|          |
 |          |           |            |<--tool---|          |
 |          |           |            |--exec--->|          |
 |          |           |            |<--result-|          |
 |          |           |            |--LLM--->|          |
 |          |           |<--response-|          |          |
 |<--resp---|           |            |          |          |
```

## 14. Pros

- **First-party from OpenAI** — direct access to the latest model features (structured outputs, responses API, streaming)
- **Minimal abstraction** — agents are thin wrappers over the OpenAI chat completions API
- **Built-in handoffs** — agent-to-agent routing without custom orchestration code
- **Guardrails** — validated input/output at the framework level, not just prompt engineering
- **OpenTelemetry tracing** — first-class observability for debugging multi-agent flows
- **Lightweight** — simple API surface, easy to learn

## 15. Cons

- **OpenAI only** — tightly coupled to OpenAI models. No support for Anthropic, Gemini, local models, etc.
- **Very new** — limited community, sparse documentation, fast API changes (released 2025)
- **No built-in RAG** — must integrate separately with vector stores for retrieval
- **No memory management** — no built-in conversation memory; you must pass history manually
- **Limited tool ecosystem** — no community tool registry (unlike LangChain's 500+ integrations)
- **No graph/state support** — no cycles or complex state machines (LangGraph is better for this)

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | Multi-provider, largest ecosystem |
| LangGraph | Stateful cyclic agent workflows |
| AutoGen | Multi-agent conversational patterns |
| Pydantic AI | Type-safe structured outputs |
| Direct OpenAI API | Simple single-agent applications |

## 17. Performance Considerations

- **LLM token usage:** The SDK uses OpenAI's latest models. `gpt-4o-mini` is ~10x cheaper than `gpt-4o`. Use mini for simple triage, 4o for complex reasoning.
- **Handoff overhead:** Each handoff requires a new LLM call. Minimise unnecessary handoffs by batching related tasks.
- **Tool execution latency:** Tools run in `on_invoke_tool`. Offload long-running tools to background workers.
- **Streaming latency:** `run_streamed` yields events as they arrive. First token latency is ~200ms for GPT-4o.
- **Guardrail performance:** Guardrails run on every input/output. Keep them lightweight (<10ms). Use async guardrails for I/O-bound checks.

## 18. Scaling to Millions

1. **Stateless agents:** Agent definitions are lightweight JSON. Scale horizontally with Kubernetes. Each pod runs agents.
2. **OpenAI API scaling:** OpenAI handles LLM scaling. Use API key rotation for higher throughput.
3. **Tool execution pool:** Run tools in a separate worker pool. Use a message queue (Redis Streams) for tool requests.
4. **Tracing pipeline:** Export traces to an OTel collector (e.g., Grafana Tempo). Sample at 1% for cost control.
5. **Caching:** Cache agent responses for identical inputs using Redis. Important: cache invalidation requires careful prompt version tracking.

## 19. Failure Scenarios

1. **OpenAI API outage:** No fallback — the SDK only works with OpenAI. Implement a circuit breaker and queue requests.
2. **Invalid handoff:** Handoff to a non-existent agent raises an error. Validate agent names at startup.
3. **Guardrail rejection:** The run returns `GuardrailResult.fail(...)`. Handle this in the application layer (return fallback response).
4. **Tool timeout:** `on_invoke_tool` must respect a timeout. Use `asyncio.wait_for` for async tools.
5. **Context window exceeded:** The SDK does not truncate history. Implement manual truncation before calling `Runner.run`.

## 20. Security

- **Tool input validation:** `params_json_schema` ensures tool arguments match the schema. Additional validation inside the tool function.
- **Guardrail enforcement:** Input guardrails prevent injection attacks. Output guardrails prevent data leakage.
- **API key security:** OpenAI API key must be in environment variables. Never expose in client-facing code.
- **Tracing data:** Traces capture full inputs/outputs. Sanitise sensitive data before exporting traces.
- **Handoff scoping:** Agents can only hand off to explicitly listed agents. No arbitrary agent routing.

## 21. Monitoring

- **OpenTelemetry traces:** Every run creates a trace with spans for agent, tool, handoff, and guardrail events.
- **Export:** Configure OTLP exporter to send traces to Grafana Tempo, Jaeger, or DataDog.
- **Metrics:** Track `agent_run_count`, `handoff_count`, `tool_invocation_count`, `guardrail_failure_rate`.
- **Logging:** Structured logs with `trace_id` for correlation. Log agent name, model, duration, token usage.

## 22. Interview Questions

1. How does the OpenAI Agents SDK handle agent-to-agent handoffs?
2. Explain the difference between input guardrails and output guardrails.
3. How would you implement a multi-agent customer support system with routing?
4. Compare the SDK's approach to tool calling with LangChain's.
5. What are the limitations of the SDK compared to LangGraph?
6. How does tracing work in the SDK? How would you export traces?
7. Design a system that uses handoffs to process a multi-step returns workflow.
8. How would you handle context management for long-running agent sessions?

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Create agent | `Agent(name="A", instructions="...", tools=[t])` |
| Add handoff | `handoffs=[other_agent]` |
| Add guardrail | `input_guardrails=[InputGuardrail(fn)]` |
| Define tool | `FunctionTool(name="t", params_json_schema=schema, on_invoke_tool=fn)` |
| Run agent | `Runner.run(agent, "hello")` |
| Stream agent | `Runner.run_streamed(agent, "hello")` |
| With tracing | `with trace("session"): result = await Runner.run(...)` |
| Access result | `result.final_output`, `result.last_agent`, `result.trace_id` |

## History

The OpenAI Agents SDK was announced and released as open-source in March 2025. It represents OpenAI's move to provide a first-party agent framework after observing the proliferation of third-party abstractions. The SDK is built on the same internal infrastructure that OpenAI uses for its own agent products (Operator, Codex CLI). The initial release focused on the core primitives — agents, handoffs, guardrails, and tracing — intentionally minimal compared to LangChain. OpenAI positioned it as the "React of agent frameworks" — a small, composable core that developers extend rather than a large framework. The SDK is Apache 2.0 licensed and accepts community contributions.

## Core Components Explained

- **Agent:** The primary abstraction. An agent has instructions (system prompt), a model (OpenAI model string), tools (functions it can call), handoffs (other agents it can delegate to), and guardrails (validation hooks). Agents can be thought of as "functions with an LLM brain."
- **Runner:** Executes an agent. `Runner.run()` processes input synchronously. `Runner.run_streamed()` yields streaming events. `Runner.run_until_handoff()` stops at the first handoff boundary.
- **FunctionTool:** A tool definition with a name, description, JSON schema for arguments, and an async function. The schema is automatically used by the LLM for structured tool calls.
- **Handoff:** Built-in mechanism for agent routing. The SDK automatically generates a `transfer_to_X` tool for each handoff target. Handoffs carry the full conversation history.
- **Guardrails:** Hooks that run before agent processing (input) or after response generation (output). Guardrails can fail, warn, or pass. Failed guardrails can redirect the conversation flow.
- **Trace:** OpenTelemetry-based instrumentation. Every agent run creates a trace with spans for all sub-operations. Exportable to any OTLP-compatible backend.
- **RunResult:** Contains the final output, the last agent that responded, the full message history, and the trace ID for observability.

## Installation

```bash
pip install openai-agents
# Requires OpenAI Python SDK
pip install openai >=1.0.0
# API key
export OPENAI_API_KEY=sk-...
```

## Advanced Features

- **Streaming events:** `run_streamed` yields granular events: `agent_response_chunk`, `tool_call`, `tool_result`, `handoff`, `guardrail_result`.
- **Context passing:** Each `Runner.run` accepts a `context` dict. Context is passed to all tools, guardrails, and handoff handlers.
- **Agentic context sharing:** Handoffs automatically share context between agents. Useful for passing customer IDs, authentication tokens.
- **Custom handoff data:** `on_handoff` callback returns a dict that is merged into the receiving agent's context.
- **Nested traces:** `trace()` contexts can be nested. A parent trace contains child traces for each agent run.
- **Tool result formatting:** Tools can return strings, dicts, or Pydantic models. The SDK serialises to JSON for the LLM.
- **Model settings:** Per-agent `model_settings` for temperature, top_p, max_tokens, stop sequences, response format.

## Performance Tuning

1. **Model selection:** Use `gpt-4o-mini` for triage/routing agents (fast, cheap). Use `gpt-4o` for specialist agents requiring deep reasoning.
2. **Tool call reduction:** Combine multiple small tools into one larger tool to reduce LLM round-trips.
3. **Streaming for UX:** Always use `run_streamed` for chat applications. Users see token-by-token output.
4. **Guardrail batching:** Run multiple lightweight guardrails in parallel within a single guardrail function.
5. **Context minimisation:** Keep context dict small. Large contexts are included in every LLM call.

## Debugging Tips

1. Inspect `result.messages` to see the full conversation including system prompts, tool calls, and handoffs.
2. Use `trace_id` to find the run in your OTel backend. Every span shows duration, input, and output.
3. Set `log_level=DEBUG` on the OpenAI client to see raw API calls.
4. Check `result.last_agent` to confirm the correct agent handled the request.
5. Common errors: `ValueError: Agent not found` (typo in handoff agent name), `GuardrailFailureError` (guardrail rejected the output).

## Production Deployment

1. **Containerise:** Docker container with the agents service. Expose as FastAPI endpoints.
2. **OpenAI API key management:** Use environment variables or a secrets manager. Rotate keys regularly.
3. **Tracing backend:** Deploy Grafana Tempo or Jaeger for trace storage. Configure sampling rate (0.1% for high-throughput).
4. **Rate limiting:** Implement per-API-key rate limiting at the gateway. OpenAI's API has its own rate limits.
5. **Graceful degradation:** On OpenAI API error, queue the request for retry. Return a fallback response (e.g., "Our AI system is temporarily unavailable").
6. **CI/CD:** Run unit tests with mock OpenAI API responses. Integration tests with a real API key in staging.
