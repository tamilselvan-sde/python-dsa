# Tool Calling / Function Calling

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Tool calling (function calling) is an LLM capability where the model outputs structured JSON requests to invoke external functions. Rather than generating text, the model decides which tool to use, with what arguments, and delegates execution to the runtime. The result is fed back into the conversation loop.

## 2. Why do we need it?
LLMs are frozen at training time — they can't access real-time data, execute code, or interact with APIs. Tool calling bridges this gap: agents can query databases, call web APIs, run calculations, send emails, and control systems. It transforms LLMs from pure text generators into interactive problem solvers.

## 3. Real-world Example
**A travel booking agent** uses tool calling to: (1) `search_flights(origin, dest, date)` — calls Amadeus API, (2) `check_weather(city, date)` — calls Weather API, (3) `book_hotel(hotel_id, dates)` — calls Booking.com API, (4) `send_confirmation(email, details)` — calls SendGrid. Each tool is a registered function with typed parameters.

## 4. Architecture Diagram (ASCII)
```
+------------------+
|    LLM Model      |<---- Tool definitions (OpenAPI spec)
+------------------+
  |  Tool Call (JSON)
  v
+------------------+
| Tool Orchestrator |
+------------------+
  |     |     |
  v     v     v
+----+ +----+ +----+
|Tool1| |Tool2| |Tool3|
|(API)| |(DB) | |(Code|
+----+ +----+ +----+
  |     |     |
  v     v     v
+------------------+
| Result Schemas   |
+------------------+
```

## 5. Internal Working
The LLM receives a list of tool definitions in the system prompt (function name, description, parameter schema). When the model decides a tool should be called, it outputs a `tool_calls` object with function name and JSON arguments (not executes anything). The runtime validates the args against the schema, executes the actual function, and sends the result back to the LLM as a new message. The LLM then incorporates the result into its response.

## 6. Production Flow
```
1. System prompt includes tool definitions
2. LLM generates response (may include tool_calls)
3. Runtime validates tool_calls structure
4. Runtime looks up function in registry
5. Runtime validates arguments against JSON Schema
6. Runtime executes function with timeout
7. Runtime captures result (or error)
8. Result sent back to LLM as tool response
9. LLM continues reasoning
```

## 7. HLD
```
+------------------+     +------------------+
|  Client App       |     |  Tool Registry   |
+------------------+     |  (Dynamic load)  |
         |               +------------------+
         v                       ^
+------------------+             |
|  Agent Runtime    |------------+
|  (FastAPI)        |
+------------------+
  |     |     |
  v     v     v
+--+ +----+ +---+
|Ext| |Safe| |API|
|Svc| |Code| |GW |
+---+ +----+ +---+
```

## 8. LLD
```python
from pydantic import BaseModel, Field, ValidationError
from typing import Any, Callable, Awaitable
from datetime import datetime
import json
import inspect

class ToolDefinition(BaseModel):
    name: str
    description: str
    parameters: dict  # JSON Schema
    required: list[str] = Field(default_factory=list)

class ToolCall(BaseModel):
    id: str
    type: str = "function"
    function: dict  # {"name": ..., "arguments": ...}
    
    @property
    def name(self) -> str:
        return self.function["name"]
    
    @property
    def arguments(self) -> dict:
        return json.loads(self.function["arguments"])

class ToolResult(BaseModel):
    tool_call_id: str
    result: Any = None
    error: str | None = None
    duration_ms: int = 0

class Tool(BaseModel):
    definition: ToolDefinition
    function: Callable[..., Awaitable[Any]]
    timeout: int = 30
    retry_count: int = 2

    class Config:
        arbitrary_types_allowed = True

    async def execute(self, **kwargs) -> ToolResult:
        start = datetime.utcnow()
        for attempt in range(self.retry_count + 1):
            try:
                result = await asyncio.wait_for(
                    self.function(**kwargs), 
                    timeout=self.timeout
                )
                duration = (datetime.utcnow() - start).total_seconds() * 1000
                return ToolResult(tool_call_id="", result=result, duration_ms=int(duration))
            except asyncio.TimeoutError:
                if attempt == self.retry_count:
                    return ToolResult(tool_call_id="", error="Tool timeout", duration_ms=self.timeout * 1000)
            except Exception as e:
                if attempt == self.retry_count:
                    return ToolResult(tool_call_id="", error=str(e), duration_ms=0)

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool):
        self._tools[tool.definition.name] = tool

    def register_from_function(self, fn: Callable, name: str = None, description: str = None):
        name = name or fn.__name__
        sig = inspect.signature(fn)
        
        properties = {}
        required = []
        for param_name, param in sig.parameters.items():
            param_type = param.annotation if param.annotation != inspect.Parameter.empty else str
            json_type = self._pytype_to_json(param_type)
            properties[param_name] = {"type": json_type, "description": f"Parameter {param_name}"}
            if param.default == inspect.Parameter.empty:
                required.append(param_name)
        
        definition = ToolDefinition(
            name=name,
            description=description or fn.__doc__ or "",
            parameters={"type": "object", "properties": properties},
            required=required
        )
        self._tools[name] = Tool(definition=definition, function=fn)

    def get_openai_tools(self) -> list[dict]:
        return [{
            "type": "function",
            "function": tool.definition.model_dump()
        } for tool in self._tools.values()]

    async def execute_tool(self, tool_call: ToolCall) -> ToolResult:
        tool = self._tools.get(tool_call.name)
        if not tool:
            return ToolResult(tool_call_id=tool_call.id, error=f"Tool '{tool_call.name}' not found")
        return await tool.execute(**tool_call.arguments)

    def _pytype_to_json(self, pytype) -> str:
        mapping = {
            str: "string", int: "integer", float: "number",
            bool: "boolean", list: "array", dict: "object"
        }
        return mapping.get(pytype, "string")
```

## 9. Python Implementation
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import httpx
from openai import AsyncOpenAI
import asyncio

app = FastAPI()
registry = ToolRegistry()

# --- Tool Implementations ---

async def search_web(query: str, num_results: int = 5) -> str:
    """Search the web and return top results"""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.serpapi.com/search",
            params={"q": query, "num": num_results, "api_key": "sk-..."}
        )
        response.raise_for_status()
        return response.text

async def calculate(expression: str) -> str:
    """Evaluate a mathematical expression"""
    try:
        # Sandboxed in production!
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

async def get_weather(city: str, country: str = "US") -> str:
    """Get current weather for a city"""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"https://api.openweathermap.org/data/2.5/weather",
            params={"q": f"{city},{country}", "appid": "sk-..."}
        )
        return resp.text

# Register tools
registry.register_from_function(search_web)
registry.register_from_function(calculate)
registry.register_from_function(get_weather)

# --- Agent with Tool Calling ---

class ToolCallAgent:
    def __init__(self, llm: AsyncOpenAI, registry: ToolRegistry, model: str = "gpt-4o"):
        self.llm = llm
        self.registry = registry
        self.model = model

    async def run(self, messages: list[dict], max_turns: int = 10) -> str:
        msgs = messages.copy()
        
        for turn in range(max_turns):
            response = await self.llm.chat.completions.create(
                model=self.model,
                messages=msgs,
                tools=self.registry.get_openai_tools(),
                tool_choice="auto",
                parallel_tool_calls=True
            )
            
            msg = response.choices[0].message
            msgs.append({
                "role": "assistant",
                "content": msg.content,
                "tool_calls": msg.model_dump().get("tool_calls", [])
            })
            
            if not msg.tool_calls:
                return msg.content or "Done"
            
            # Execute all tool calls in parallel
            tool_tasks = []
            for tc in msg.tool_calls:
                tool_call = ToolCall(id=tc.id, function={
                    "name": tc.function.name,
                    "arguments": tc.function.arguments
                })
                tool_tasks.append(self.registry.execute_tool(tool_call))
            
            results = await asyncio.gather(*tool_tasks)
            
            for tc, result in zip(msg.tool_calls, results):
                msgs.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": json.dumps(result.result) if not result.error else result.error
                })
        
        msgs.append({"role": "user", "content": "Provide a final summary."})
        final = await self.llm.chat.completions.create(model=self.model, messages=msgs)
        return final.choices[0].message.content or "Max turns reached without final answer"


@app.post("/agent/run")
async def run_agent(request: dict):
    agent = ToolCallAgent(llm=AsyncOpenAI(), registry=registry)
    result = await agent.run(messages=request.get("messages", []))
    return {"result": result}
```

## 10. Folder Structure
```
tools/
├── registry/
│   ├── registry.py    # ToolRegistry class
│   ├── decorators.py  # @tool decorator for registration
│   └── validation.py  # JSON Schema argument validation
├── builtins/
│   ├── web_search.py
│   ├── calculator.py
│   ├── code_executor.py
│   ├── file_ops.py
│   └── email_sender.py
├── integrations/
│   ├── slack.py
│   ├── jira.py
│   ├── github.py
│   ├── databases/
│   │   ├── postgres.py
│   │   └── redis.py
│   └── apis/
│       ├── stripe.py
│       ├── sendgrid.py
│       └── twilio.py
├── sandbox/
│   ├── container.py    # gVisor/Firecracker sandbox
│   └── policies.py     # Tool access control
├── api/
│   └── routes.py
└── main.py
```

## 11. Configuration
```yaml
tool_calling:
  parallel_execution: true
  max_tools_per_call: 5
  default_timeout: 30
  default_retries: 2
  
  sandbox:
    enabled: true
    runtime: gvisor
    memory_limit_mb: 256
    cpu_limit: 0.5
    network_access: restricted
    
  rate_limits:
    web_search: 10/minute
    email_send: 5/minute
    code_exec: 20/minute
  
  logging:
    log_all_inputs: true
    log_all_outputs: true
    max_result_size: 100000
```

## 12. Flowchart
```
[LLM generates tool_call]
        |
   [Validate structure]
        |
   [Validate args (JSON Schema)]
        |
   [Rate limit check] --Fail--> [Return error]
        |
        Pass
        |
   [Execute function with timeout]
        |
   [Success?] --No--> [Retry?] --Yes--> [Retry]
        |                 No               |
        Yes                |               |
        |                  v               |
   [Return result]   [Return error]        |
        |                  |               |
        +--------+---------+---------------+
                 |
        [LLM receives result]
```

## 13. Sequence Diagram
```
LLM            ToolOrchestrator     Tool1          Tool2
 |                    |                |              |
 |--tool_calls------->|                |              |
 |                    |                |              |
 |                    |--validate------|              |
 |                    |  args          |              |
 |                    |                |              |
 |                    |--execute(par)--|              |
 |                    |                |              |
 |                    |--execute(par)---------------->|
 |                    |                |              |
 |                    |<--result1------|              |
 |                    |<--result2--------------------|
 |                    |                |              |
 |                    |--rank & merge--|              |
 |                    |  results       |              |
 |                    |                |              |
 |<--tool_results-----|                |              |
 |                    |                |              |
 |--final_response    |                |              |
```

## 14. Pros
- LLM handles complex routing logic (which tool, what args)
- Parallel tool execution possible
- Rich ecosystem — any function can be a tool
- Structured output from LLM (not free-form)
- Caching and reuse of tool definitions

## 15. Cons
- LLM must be trained for tool calling (not all models support it)
- JSON schema drift between definition and implementation
- LLM hallucinates tool arguments (especially for complex schemas)
- Security surface area increases with each tool
- Cost per turn (tool call = extra LLM call for result processing)

## 16. Alternatives
- **Text-based tool use** — model describes action in text, parsed by regex
- **API chaining** — hardcoded pipelines, no LLM routing
- **Semantic kernel** — MSFT's approach to AI orchestration
- **LangChain tools** — abstraction layer on top of tool calling

## 17. Performance Considerations
- Parallel tool calls reduce wall-clock time (await asyncio.gather)
- Tool definitions add tokens to context (~500 tokens per tool)
- Keep tool descriptions concise but unambiguous
- Cache tool results for pure functions (calculate: `(2+2)` always = 4)
- Timeout each tool independently — don't let one slow tool block others

## 18. Scaling to Millions
```
Tool Execution Pool:
  - 50+ worker pods for tool execution
  - Tools with GPU needs (code exec) scheduled separately
  - Queue-based execution for slow tools (email, SMS)
  - Result cache (Redis) for deterministic tools (TTL = 300s)

Rate Limiting:
  - Distributed token buckets (Redis Lua scripts)
  - Per-tool, per-user, per-tenant limits
  - Burst capacity on critical path tools

Definition Distribution:
  - Tool definitions served from CDN (static JSON)
  - Versioned tool schemas (no breaking changes without migration)
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Tool timeout | Blocked agent | asyncio.wait_for with default 30s |
| Invalid args | Tool rejection | LLM retry with error message |
| API key expired | Auth failure | Health check, auto-rotation |
| Rate limit hit | Throttled tool | Queue + retry with delay |
| LLM ignores tool | Prose response | tool_choice="required" |

## 20. Security
- Never expose tools that execute raw SQL or shell commands
- Sandbox code execution tools (gVisor, Firecracker microVMs)
- Validate all args client-side AND server-side
- Rate limit per tool to prevent abuse
- Audit log every tool call with caller identity
- Tool registry whitelist — no dynamic tool loading
- Secret injection at runtime, not in tool definitions

## 21. Monitoring
```prometheus
tool_calls_total{name="web_search|calculate|send_email", status="success|error"}
tool_latency_seconds{name="*", p50="", p95="", p99=""}
tool_error_rate_by_code{name="*"}
tool_rate_limit_hits_total{name="*"}
tool_result_size_bytes{name="*"}
tool_calls_parallel_batch_size
```

## 22. Interview Questions
**Q1**: How do you handle tool calling when the LLM provides invalid arguments? (Send error back as tool response with validation details; let LLM self-correct)

**Q2**: Can parallel tool calls have dependencies? (No — true parallel requires independence. Use Plan-and-Execute for dependent tool chains)

**Q3**: How would you make tool definitions available to the model efficiently? (Cache definitions, use concise but unique descriptions, avoid redundant tools)

## 23. Cheat Sheet
```
Tool Def = {"name", "description", "parameters": JSON Schema, "required"}
LLM Output = {"tool_calls": [{"id", "function": {"name", "arguments"}}]}
Result = {"role": "tool", "tool_call_id": ..., "content": ...}
Best Practice: parallel_tool_calls=True, but max 5 per turn
Timeout per tool: 10-30s
Retry: 2-3 times for transient failures
```

## 24. Common Mistakes
- Overly long tool descriptions (waste tokens)
- Not setting `tool_choice` — model may ignore tools
- Registering dozens of tools — model confuses similar ones
- No timeout → tool blocks forever
- No result size limit → massive JSON in context
- Allowing dangerous tools (shell exec, SQL write) without guardrails
- Not handling tool_call_id mapping correctly in multi-call responses

## 25. Production Best Practices
1. Always validate arguments on the server side, not trusting LLM
2. Use parallel execution when tools are independent
3. Set `max_result_size` — truncate huge responses
4. Log every tool input/output for debugging
5. Implement circuit breakers for flaky tools (3 failures in 60s = suspend 5min)
6. Version tool schemas — breaking changes require migration period
7. Test with tool_choice="required" to verify tool calling works end-to-end
8. Cache deterministic tool results (calculate, format, lookup)
