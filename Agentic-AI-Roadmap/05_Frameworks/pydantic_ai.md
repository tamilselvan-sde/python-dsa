# Pydantic AI

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

Pydantic AI is an open-source Python agent framework built on top of Pydantic (the data validation library) by Samuel Colvin (creator of Pydantic). Released in 2024, it provides a type-safe, structured approach to building LLM-powered applications. The framework is designed around Pydantic's core philosophy — **validate everything** — ensuring that LLM inputs and outputs are rigorously type-checked. Pydantic AI supports OpenAI, Anthropic, Gemini, Groq, Mistral, and Cohere models.

## 2. Why do we need it?

LLM outputs are notoriously unstructured and unreliable. Existing frameworks treat LLM output as freeform text, forcing developers to hand-parse and validate. Pydantic AI solves:
- **Type safety:** Every LLM response is validated against a Pydantic model at compile time
- **Structured outputs:** Define exactly what you expect the LLM to return — JSON, Python objects, or complex nested schemas
- **Validation-driven development:** Define the output schema first, then build prompts to match
- **Dependency injection:** Agents have typed dependencies, making testing straightforward
- **Streaming with validation:** Stream tokens while validating against the schema

## 3. Real-world Example

**Product review classifier:** An e-commerce platform needs to extract structured data from customer reviews. Pydantic AI defines a `ReviewAnalysis` model (sentiment, rating, key_phrases, category). An agent prompts the LLM with the review text and validates the response against the model. Invalid outputs are automatically retried with the validation error as feedback.

## 4. Architecture Diagram (ASCII)

```
                    +-------------------------+
                    |     Pydantic AI         |
                    +-------------------------+
                    |                         |
    +---------------+    +--------------------+---------------+
    |  Agent        |    |    Tool                           |
    |  (orchestr.)  |    |  (function + schema)             |
    +-------+-------+    +-------------------+---------------+
            |                                |
    +-------v-------+          +-------------v-----------+
    |  Model         |          |  Result Validation     |
    |  (OpenAI, etc) |          |  (Pydantic model)      |
    +-------+-------+          +-------------+-----------+
            |                                |
    +-------v---------------+                |
    |  System Prompt        |                |
    |  + Result Type        |                |
    +-----------------------+                |
                                   +---------v----------+
                                   |  Retry with Error  |
                                   |  Message            |
                                   +--------------------+
```

## 5. Internal Working

Pydantic AI's execution flow is fundamentally different from other frameworks:

1. **Agent definition:** An `Agent` is parametrised with a `result_type` (Pydantic model or TypeVar), `model` (LLM provider), `system_prompt`, and `tools`.
2. **Tool registration:** Tools are Python functions with Pydantic-validated arguments. The tool schema is automatically derived from the function signature.
3. **Execution:** 
   - User sends a message
   - Agent constructs a prompt with system prompt, conversation history, and tool schemas
   - LLM produces a response (tool call or final message)
   - If tool call: tool executes, result is validated, appended to history, loop continues
   - If final message: Pydantic validates the output against `result_type`
   - If validation fails: error message is appended to history, LLM retries
4. **Result:** A `RunResult` object with validated `data`, `usage` (token counts), and `history`.

## 6. Production Flow

```
User Message → Agent receives → Format prompt (system + history + tools) →
LLM generates response → Is it a tool call? → Yes: Execute tool → validate →
loop back → No: Validate final output against result_type →
Validated result (Pydantic model) → Return to user
```

## 7. HLD

```
                   +---------------------+
                   |   API Gateway       |
                   +---------+-----------+
                             |
                   +---------+-----------+
                   |   Pydantic AI       |
                   |   Service           |
                   +---------+-----------+
                             |
              +--------------+--------------+
              |              |              |
    +---------v----+  +-----v------+  +----v---------+
    | LLM Provider |  | Agent Pool |  | Result Store |
    | (OpenAI)     |  | (pre-ini)  |  | (Postgres)   |
    +--------------+  +------------+  +--------------+
```

## 8. LLD

```python
Agent[ResultType]
  ├── model: Union[OpenAI, Anthropic, Gemini, ...]
  ├── result_type: Type[ResultType]  # Pydantic model
  ├── system_prompt: str | Callable
  ├── tools: list[Tool]
  ├── deps_type: Type[Deps]  # dependency injection
  ├── max_retries: int
  ├── run(user_message) -> RunResult[ResultType]
  ├── run_stream(user_message) -> StreamRunResult
  └── run_sync(user_message) -> SyncRunResult

Tool
  ├── name: str
  ├── description: str
  ├── args_schema: Type[BaseModel]
  └── fn: Callable  # the actual implementation

RunResult[ResultType]
  ├── data: ResultType  # validated Pydantic model
  ├── usage: Usage  # token counts
  ├── history: list[ModelMessage]
  └── new_messages: list[ModelMessage]

Usage
  ├── request_tokens: int
  ├── response_tokens: int
  └── total_tokens: int
```

## 9. Python Implementation

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext, Tool
from pydantic_ai.models.openai import OpenAIModel
from dataclasses import dataclass
import asyncio

# Define the structured output
class ReviewAnalysis(BaseModel):
    sentiment: str = Field(description="Positive, Negative, or Neutral")
    rating: int = Field(ge=1, le=5, description="1-5 star rating")
    key_phrases: list[str] = Field(description="Important phrases from the review")
    category: str = Field(description="Product category: electronics, clothing, food, other")
    is_fake_review: bool = Field(default=False, description="Suspected fake review")

# Define dependencies
@dataclass
class ReviewDeps:
    user_id: str
    product_db: dict  # simplified

# Define tools with typed dependencies
async def lookup_product(ctx: RunContext[ReviewDeps], product_name: str) -> str:
    """Look up product details from the database."""
    product_db = ctx.deps.product_db
    return str(product_db.get(product_name, "Unknown product"))

# Create agent with type-safe result
model = OpenAIModel("gpt-4o", api_key="sk-...")
agent = Agent[ReviewDeps, ReviewAnalysis](
    model=model,
    result_type=ReviewAnalysis,
    system_prompt=(
        "You are a product review analyst. "
        "Extract structured information from customer reviews. "
        "Be thorough and accurate. Detect potential fake reviews."
    ),
    deps_type=ReviewDeps,
    tools=[Tool(lookup_product)],
    max_retries=3,
)

# Run the agent
async def main():
    deps = ReviewDeps(
        user_id="user_123",
        product_db={"laptop_x": "High-end gaming laptop", "phone_y": "Budget smartphone"}
    )

    result = await agent.run(
        "I bought the laptop X and it's amazing! The screen is gorgeous, "
        "battery lasts all day. 5 stars totally worth it.",
        deps=deps,
    )

    print(f"Sentiment: {result.data.sentiment}")
    print(f"Rating: {result.data.rating}")
    print(f"Key phrases: {result.data.key_phrases}")
    print(f"Category: {result.data.category}")
    print(f"Fake: {result.data.is_fake_review}")
    print(f"Tokens used: {result.usage().total_tokens}")

    # Structured access
    analysis = result.data
    if analysis.rating < 2 and analysis.sentiment == "Positive":
        print("Warning: Sentiment/rating mismatch detected!")

asyncio.run(main())
```

### Streaming Example

```python
async for chunk in agent.run_stream("This product is terrible..."):
    # Streams tokens while validating against ReviewAnalysis
    if chunk.is_partial:
        print(chunk.data.model_dump())  # partial validation
    else:
        print("Final:", chunk.data)
```

## 10. Folder Structure

```
pydantic-ai-app/
├── agents/
│   ├── __init__.py
│   ├── review_agent.py
│   ├── support_agent.py
│   └── data_extraction_agent.py
├── models/
│   ├── __init__.py
│   ├── schemas.py          # Pydantic result models
│   └── deps.py             # Dependency dataclasses
├── tools/
│   ├── __init__.py
│   ├── search_tool.py
│   └── database_tool.py
├── api/
│   └── main.py
├── tests/
│   ├── test_agents.py
│   └── test_models.py
├── pyproject.toml
└── .env
```

## 11. Configuration

```python
from pydantic_settings import BaseSettings

class PydanticAIConfig(BaseSettings):
    openai_api_key: str = ""
    anthropic_api_key: str = ""
    gemini_api_key: str = ""
    default_model: str = "openai:gpt-4o"
    max_retries: int = 3
    log_level: str = "INFO"

    class Config:
        env_file = ".env"

config = PydanticAIConfig()
```

## 12. Flowchart

```
User Message
    |
    v
Agent.run()
    |
    v
Format Messages (system + history + schema)
    |
    v
LLM Call
    |
    v
Parse Response
    |
    +-- Tool Call? --> Execute Tool --> Validate Tool Output --> Append to History --> Loop
    |
    +-- Final Response?
         |
         v
    Validate against result_type
         |
    +-- Valid? --> Return RunResult[ResultType]
    |
    +-- Invalid? --> Append Validation Error to History --> Retry (max_retries times)
         |
         +-- Exhausted? --> Raise error or return partial
```

## 13. Sequence Diagram

```
User         Agent         LLM          Tool        Validator
 |            |             |            |             |
 |--run()---->|             |            |             |
 |            |--LLM call-->|            |             |
 |            |<--response--|            |             |
 |            |             |            |             |
 |            +--[Tool call?]            |             |
 |            |--execute---------------->|             |
 |            |<--result-----------------|             |
 |            |--LLM call-->|            |             |
 |            |<--response--|            |             |
 |            |             |            |             |
 |            +--[Final?]   |            |             |
 |            |--validate-->|            |             |
 |            |<--valid-----|            |             |
 |<--result---|             |            |             |
```

## 14. Pros

- **Type-safe by design** — every LLM output is validated against a Pydantic model
- **Validation-driven retry** — validation errors become LLM prompts, improving reliability
- **Dependency injection** — agents have typed deps, making testing trivial
- **Native asyncio** — first-class async support throughout
- **Clean, minimal API** — fewer concepts than LangChain, easier to reason about
- **Pydantic ecosystem** — works seamlessly with Pydantic settings, Logfire, etc.

## 15. Cons

- **Very new** — smaller community, fewer integrations (mid-2024 release)
- **No built-in RAG** — must integrate separately with vector stores
- **Limited agent orchestration** — no multi-agent or graph support (yet)
- **Pydantic-centric** — steep learning curve if unfamiliar with Pydantic v2
- **Fewer LLM providers** — growing, but not as many as LangChain

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | If you need the largest ecosystem and integrations |
| Instructor | Similar validation approach, lighter weight |
| Outlines | Structured generation via grammar constraints |
| TypeChat | Microsoft's type-safe prompts (TypeScript) |
| Marvin | AI functions with Pydantic, OpenAI-focused |

## 17. Performance Considerations

- **Validation overhead:** Pydantic validation is fast (<1ms for most models). For deeply nested models, use `model_validate` once rather than field-by-field validation.
- **Retry cost:** Each retry calls the LLM again. Set `max_retries` based on accuracy needs. 3 retries is a good default.
- **Streaming performance:** `run_stream` validates partial results. The cost of partial validation is lower than re-running the full agent.
- **Token usage:** Pydantic AI includes the full schema in the system prompt. Large schemas consume more tokens. Keep result models focused.
- **Tool argument validation:** Tools automatically validate arguments via Pydantic. Complex arguments add <1ms overhead.

## 18. Scaling to Millions

1. **Stateless agents:** Agent definitions are stateless. Scale horizontally with Kubernetes. Each pod runs the same agents.
2. **Result persistence:** Store `RunResult` (JSON-serialised) in PostgreSQL. Use for auditing, debugging, and retraining.
3. **Tool isolation:** Run tools in separate processes for fault isolation. Use a message queue for tool requests.
4. **Caching:** Cache LLM responses for identical inputs. Use Pydantic's hash of the entire prompt as cache key.
5. **Model routing:** Use different model tiers (fast vs. accurate) based on task complexity. Pydantic AI supports per-agent model configuration.

## 19. Failure Scenarios

1. **Validation failure after retries exhausted:** The agent raises `AgentRunError`. Catch it, log the raw LLM output, and fall back to a default response.
2. **Tool execution error:** The error message is fed back to the LLM. The LLM retries with a different approach.
3. **LLM output does not match schema:** Validation error with `ValidationError` details. The LLM sees the exact error and fixes the output.
4. **Rate limiting:** Pydantic AI does not have built-in rate limiting. Implement externally via a token bucket.
5. **Context window exceeded:** Conversation history grows. Implement sliding window truncation outside Pydantic AI.

## 20. Security

- **Schema injection:** Never include user input directly in the `result_type` schema. Keep schemas static and predefined.
- **Tool access control:** Validate that the agent's deps include proper permissions. `RunContext.deps` carries user context.
- **Input sanitisation:** Pydantic AI validates tool arguments. Malformed input is rejected before reaching the tool.
- **Logging PII:** Use Pydantic's `Field(exclude=True)` to prevent sensitive fields from being logged.
- **Data validation:** Pydantic's strict mode (`strict=True`) rejects type coercion, preventing injection attacks.

## 21. Monitoring

- **Logfire:** Pydantic's observability platform. Logfire captures agent runs, LLM calls, tool executions, and validation events.
- **Custom logging:** `Agent` accepts an `on_agent_error` callback. Log `RunResult.usage()` for token tracking.
- **Metrics:** Track `agent_run_count`, `validation_success_rate`, `retry_count`, `tool_latency_ms`.
- **Alerts:** Alert on `retry_rate > 10%` (indicates schema issues) or `validation_failure_rate > 5%`.

## 22. Interview Questions

1. How does Pydantic AI's validation-driven approach differ from LangChain's output parsers?
2. Explain how dependency injection works in Pydantic AI agents.
3. How does streaming with validation work? What are the trade-offs?
4. Design a Pydantic AI agent that extracts structured data from invoices.
5. How does Pydantic AI handle retries when validation fails?
6. Compare Pydantic AI's type safety with Instructor's approach.
7. How would you implement a tool that requires authentication?
8. What happens when the LLM returns output that doesn't match the result_type?

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Define agent | `Agent[MyDeps, MyResult](model, result_type=MyResult)` |
| Define result model | `class MyResult(BaseModel): field: str` |
| Run agent | `await agent.run("question", deps=my_deps)` |
| Stream agent | `async for chunk in agent.run_stream("question", deps=my_deps)` |
| Add tool | `Tool(my_function)` |
| Access result | `result.data.field` |
| Access usage | `result.usage().total_tokens` |
| Access history | `result.history` |

## History

Pydantic AI was created by Samuel Colvin, the creator of Pydantic, and first publicly released in early 2024. The framework emerged from Colvin's frustration with the lack of type safety in existing LLM frameworks. Pydantic AI builds on Pydantic v2's improved validation engine and leverages the existing Pydantic ecosystem (Logfire, Pydantic Settings). The framework gained immediate attention from the Python community for its clean API and strong typing guarantees. Version 0.2 added streaming support. By version 0.5, Pydantic AI had support for multiple model providers, tools, and dependency injection. As of 2025, Pydantic AI is rapidly growing as a preferred framework for developers who value type safety and validation-driven development.

## Core Components Explained

- **Agent:** The central orchestrator. Generic over `Deps` (dependencies) and `ResultType` (output model). Handles prompt construction, LLM calling, tool execution, and result validation.
- **Result types:** Any Pydantic `BaseModel`. The agent prompts the LLM to produce output matching this model. On validation failure, the agent retries with error messages.
- **Tool:** A Python function wrapped with a Pydantic argument schema. The schema is automatically derived from the function signature and included in the LLM prompt.
- **RunContext:** Carries typed dependencies, conversation history, and agent configuration. Passed to every tool and prompt function.
- **RunResult:** The output of a run. Contains validated `data`, token `usage`, and message `history`.
- **Models:** LLM provider wrappers (`OpenAIModel`, `AnthropicModel`, `GeminiModel`). Each handles API-specific formatting and streaming.

## Installation

```bash
pip install pydantic-ai
# With specific providers
pip install pydantic-ai[openai]
pip install pydantic-ai[anthropic]
pip install pydantic-ai[gemini]
# Full
pip install pydantic-ai[all]
```

## Advanced Features

- **Validation-driven retry:** When the LLM returns invalid output, the validation error is formatted into a new prompt. The LLM sees exactly what was wrong and fixes it.
- **Structured streaming:** `agent.run_stream()` yields partial validation results. Each chunk is validated incrementally.
- **Dependency injection:** `RunContext[Deps]` carries typed dependencies. Tools access `ctx.deps` for connection pools, user info, etc.
- **Dynamic system prompts:** `system_prompt` can be a callable that receives `RunContext` and returns a string.
- **Result validators:** Post-processing functions that validate or transform the result after LLM generation.
- **Model fallbacks:** Multiple models can be configured. If one fails, the agent tries the next.
- **Tool documentation:** Automatically generates tool descriptions from function docstrings and Pydantic field descriptions.

## Performance Tuning

1. **Simplify result models:** Large, deeply nested Pydantic models increase token usage (schema is in the prompt) and validation time. Keep models flat and focused.
2. **Reduce retries:** `max_retries=1-2` for simple tasks, `max_retries=3-5` for complex extraction. Each retry doubles latency.
3. **Streaming for long responses:** Use `run_stream` to show partial results. Users see token-by-token output while validation happens.
4. **Tool batching:** If an agent calls multiple tools, batch independent calls. Pydantic AI executes tools sequentially per agent.
5. **Caching:** Create your own `@cache` decorator for LLM calls. Hash the full prompt (system + history + tool schemas).

## Debugging Tips

1. Set `model_settings = agent.run("question", model_settings={"log_level": "DEBUG"})` for verbose logging.
2. Inspect `result.history` to see every message (system, user, assistant, tool) in the conversation.
3. Use `logfire.instrument()` for full tracing with Logfire dashboard.
4. Check `result.usage()` for token counts per model call.
5. Common errors: `pydantic_core.ValidationError` (schema mismatch), `AgentRunError` (retries exhausted).

## Production Deployment

1. **FastAPI integration:** Mount agents as FastAPI endpoints. Use `agent.run_sync()` for synchronous endpoints.
2. **Stateless scaling:** Deploy agents as stateless API servers. Redis for session persistence if needed.
3. **Logfire monitoring:** Pydantic's Logfire platform provides real-time monitoring, trace inspection, and alerting.
4. **Graceful degradation:** Wrap agent runs in try/except. On failure, return a partial result with an error flag.
5. **Schema versioning:** Version your result models (e.g., `ReviewAnalysisV1`). Run migration scripts for backward compatibility.
