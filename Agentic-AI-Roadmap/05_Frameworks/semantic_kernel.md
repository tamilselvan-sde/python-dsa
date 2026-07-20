# Microsoft Semantic Kernel

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

Semantic Kernel (SK) is an open-source SDK developed by Microsoft (released March 2023) that enables AI orchestration through **orchestrating functions, plugins, and memories** around LLMs. It provides a lightweight, enterprise-focused framework for building AI agents that can call plugins, manage state, and integrate deeply with the Microsoft ecosystem (Azure, Microsoft 365, Copilot). SK supports C#, Python, and Java — uniquely positioning it as the go-to framework for .NET enterprises.

## 2. Why do we need it?

Enterprises adopting AI need a framework that integrates with existing infrastructure (Azure AD, Microsoft Graph, .NET) and supports enterprise patterns (DI, logging, telemetry). Semantic Kernel solves:
- **Plugin ecosystem:** Integrate LLM calls with existing APIs, databases, and services
- **Function chaining:** Compose AI and traditional functions together
- **Memory & embeddings:** Built-in vector memory for RAG
- **Multi-language:** .NET, Python, Java — consistent API across stacks
- **Microsoft ecosystem:** Seamless Azure OpenAI, Copilot, and M365 integration

## 3. Real-world Example

**Enterprise Copilot for HR:** A Semantic Kernel agent connects to Microsoft Graph to read employee calendars, SharePoint for company policies, and a SQL database for payroll data. When an employee asks "How many vacation days do I have left?", the agent: 1) Authenticates via Azure AD, 2) Retrieves employee record from SQL, 3) Reads vacation policy from SharePoint, 4) Calls Azure OpenAI to compute the answer, 5) Returns the result with a citation to the policy document.

## 4. Architecture Diagram (ASCII)

```
                    +----------------------+
                    |   Semantic Kernel    |
                    +----------------------+
                    |                      |
    +---------------+    +-----------------+---------------+
    |   Kernel      |    |     Planner                  |
    |   (orchestr.) |    |  (auto-function chaining)    |
    +------+--------+    +---------------+-------------+
           |                             |
    +------v--------+        +-----------v-----------+
    |   Memory      |        |    AI Service         |
    |  (embeddings) |        |  (OpenAI / Azure)     |
    +------+--------+        +-----------------------+
           |
    +------v--------+
    |  Plugins      |
    |  (native +    |
    |   semantic)   |
    +------+--------+
           |
    +------v--------+
    |  Connectors   |
    |  (Graph, SQL, |
    |   REST, etc.) |
    +---------------+
```

## 5. Internal Working

Semantic Kernel's core is the **Kernel** object:

1. **AI Services**: Register LLM connections (Azure OpenAI, OpenAI, Hugging Face)
2. **Plugins:** Collections of functions. Two types:
   - **Semantic functions:** Defined by a prompt template (in `skprompt.txt` + `config.json`)
   - **Native functions:** Standard C#/Python methods decorated with `[KernelFunction]`
3. **Planner:** Automatically chains functions into a plan to fulfill a user's ask. The planner uses the LLM to select and sequence functions.
4. **Memory:** Vector storage for semantic memory. Supports Azure Cognitive Search, Chroma, Qdrant, and in-memory.
5. **Filters:** Middleware-like hooks for function invocation, prompt rendering, and response handling — enabling telemetry, guardrails, and custom logic.

## 6. Production Flow

```
User Input → Kernel receives → Planner decomposes goal →
Select functions from plugins → Execute each function →
   (AI call → native code → AI call → ...) →
Collect results → Return to user → Filters log every step
```

## 7. HLD

```
                   +----------------------+
                   |   API Gateway        |
                   +----------+-----------+
                              |
                   +----------+-----------+
                   |   Semantic Kernel    |
                   |   Service            |
                   +----------+-----------+
                              |
    +-------------------------+--------------------------+
    |                         |                          |
+---v----------+    +--------v-------+    +-------------v--+
| Azure OpenAI |    | Plugin Host    |    | Memory Store   |
| (LLM)        |    | (.NET runtime) |    | (Azure Search) |
+--------------+    +--------+-------+    +----------------+
                             |
                 +-----------v-----------+
                 | Microsoft Graph,      |
                 | SQL Server, REST APIs |
                 +-----------------------+
```

## 8. LLD

```
Kernel
  ├── AddOpenAIChatCompletion(...)
  ├── ImportPluginFromDirectory(...)
  ├── ImportPluginFromObject(...)
  ├── CreateFunctionFromPrompt(...)
  └── InvokeAsync(function, variables)

KernelFunction
  ├── Name, Description
  ├── ReturnParameter
  └── InvokeAsync(kernel, arguments)

KernelPlugin
  ├── Name
  ├── FunctionCount
  └── [KernelFunction] methods

Memory
  ├── SaveInformationAsync(collection, text, id, description)
  ├── SearchAsync(collection, query, limit)
  └── SemanticTextMemory(MemoryStore)

Planner
  ├── HandlebarsPlanner
  ├── SequentialPlanner
  └── FunctionRunResult

Filters
  ├── IFunctionFilter  (OnFunctionInvoking, OnFunctionInvoked)
  ├── IPromptFilter    (OnPromptRendering, OnPromptRendered)
  └── IContentFilter   (OnResponseReceiving)
```

## 9. Python Implementation

```python
import asyncio
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.connectors.ai.function_call_behavior import FunctionCallBehavior
from semantic_kernel.plugins.core import PromptTemplateConfig
from semantic_kernel.functions import kernel_function

# Create kernel
kernel = Kernel()

# Add AI service
kernel.add_service(
    AzureChatCompletion(
        deployment_name="gpt-4o",
        endpoint="https://my-openai.openai.azure.com/",
        api_key="..."
    )
)

# Native plugin (Python class with kernel_function decorator)
class CalendarPlugin:
    @kernel_function(
        description="Get the current date and time",
        name="get_current_datetime"
    )
    def get_current_datetime(self) -> str:
        from datetime import datetime
        return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    @kernel_function(
        description="Check calendar availability for a given date",
        name="check_availability"
    )
    def check_availability(self, date: str) -> str:
        # Call calendar API
        return f"Available slots on {date}: 9am, 10am, 2pm"

# Import plugin
calendar_plugin = kernel.add_plugin(CalendarPlugin(), plugin_name="calendar")

# Invoke a function directly
result = asyncio.run(
    kernel.invoke(calendar_plugin["get_current_datetime"])
)
print(f"Current time: {result}")

# Auto function calling (planner)
from semantic_kernel.connectors.ai.open_ai.prompt_execution_settings.azure_chat_prompt_execution_settings import (
    AzureChatPromptExecutionSettings,
)

settings = kernel.get_prompt_execution_settings_from_service_id(
    "default",
    AzureChatPromptExecutionSettings
)
settings.function_call_behavior = FunctionCallBehavior.AutoInvokeKernelFunctions(
    max_number_of_auto_invoke_attempts=5
)

from semantic_kernel.functions import KernelArguments

result = asyncio.run(
    kernel.invoke_prompt(
        function_name="chat",
        plugin_name="ChatPlugin",
        prompt="What time is it and when is the next available meeting slot?",
        arguments=KernelArguments(settings=settings)
    )
)
print(result)
```

## 10. Folder Structure

```
semantic-kernel-app/
├── sk-chat-app/
│   ├── app.py
│   └── config.py
├── plugins/
│   ├── CalendarPlugin/
│   │   └── __init__.py          # Native functions
│   ├── EmailPlugin/
│   │   └── __init__.py
│   └── SummarizePlugin/
│       ├── skprompt.txt          # Semantic function prompt
│       └── config.json           # Prompt config (temperature, etc.)
├── memory/
│   └── memory_store.py
├── tests/
├── .env
├── requirements.txt
└── Dockerfile
```

## 11. Configuration

```python
# config.py
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion, AzureTextEmbedding

def configure_kernel() -> Kernel:
    kernel = Kernel()

    kernel.add_service(
        AzureChatCompletion(
            deployment_name="gpt-4o",
            endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OPENAI_KEY"),
            service_id="default"
        )
    )

    kernel.add_service(
        AzureTextEmbedding(
            deployment_name="text-embedding-3-small",
            endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OPENAI_KEY"),
            service_id="embeddings"
        )
    )

    return kernel
```

## 12. Flowchart

```
         User Goal
             |
     +-------v--------+
     |   Planner      |
     | (LLM generates |
     |  a plan)       |
     +-------+--------+
             |
    +--------+--------+
    |    Plan Steps   |
    | Step 1: Function A
    | Step 2: Function B
    | Step 3: Function C
    +--------+--------+
             |
    +--------v--------+
    | Execute Step 1  |
    +--------+--------+
             |
    +--------v--------+
    | Filter: Before  |  (IFunctionFilter)
    +--------+--------+
             |
    +--------v--------+
    | Run Function A  |
    +--------+--------+
             |
    +--------v--------+
    | Filter: After   |
    +--------+--------+
             |
    +--------+--------+ (repeat for each step)
             |
    +--------v--------+
    |   Return Result |
    +-----------------+
```

## 13. Sequence Diagram

```
User        Kernel       Planner      Plugin A    Plugin B    Azure OpenAI
 |            |            |             |           |             |
 |--ask------>|            |             |           |             |
 |            |--plan----->|             |           |             |
 |            |            |--LLM call-------------->|             |
 |            |            |<--plan-------|           |             |
 |            |--execute-->|             |           |             |
 |            |            |--invoke---->|           |             |
 |            |            |             |--LLM---->|             |
 |            |            |<--result----|           |             |
 |            |            |--invoke---->|           |             |
 |            |            |<--result----|           |             |
 |<--answer---|            |             |           |             |
```

## 14. Pros

- **Deep .NET integration** — native DI, logging, configuration, telemetry
- **Multi-language** — consistent API across C#, Python, Java
- **Plugin-first architecture** — decouples AI logic from business logic
- **Microsoft ecosystem** — seamless Azure OpenAI, Graph, Copilot, M365
- **Enterprise-ready** — filters for governance, telemetry, guardrails
- **Strong typing** — compile-time checks for function schemas

## 15. Cons

- **Python support lags** — Python SDK is less mature than C# version
- **Smaller community** — less adoption than LangChain or LlamaIndex
- **Planner limitations** — complex plans fail or produce unreliable sequences
- **Windows-centric** — some integrations assume Windows/Azure environment
- **Fast-moving API** — breaking changes across minor versions

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | Largest ecosystem, model-agnostic |
| AutoGen | Multi-agent conversational patterns |
| Bot Framework | Traditional chatbots, no planning |
| Prompt flow | Visual prompt engineering, Azure ML |

## 17. Performance Considerations

- **Plugin invocation overhead:** Each plugin call adds serialisation/deserialisation. Keep plugin arguments small.
- **Memory searches:** Use Azure Cognitive Search for production; InMemory for dev only.
- **Planner complexity:** Sequential planner calls the LLM once. Handlebars planner is more flexible but slower.
- **Filter overhead:** Multiple filters add serial latency. Keep filters lightweight and async.
- **Streaming:** SK supports streaming for chat completions. Use `streaming=True` for responsive UI.

## 18. Scaling to Millions

1. **Kernel pooling:** Pre-create a pool of Kernel instances. Each request gets a Kernel from the pool.
2. **Plugin isolation:** Run plugins (especially native code) in separate processes for fault isolation.
3. **Memory store scaling:** Azure Cognitive Search scales to billions of vectors. Configure sharding for high throughput.
4. **Planner result caching:** Cache common plans (the LLM plan generation is expensive). Invalidate on plugin changes.
5. **Observability at scale:** Use Azure Application Insights for distributed tracing across plugins and LLM calls.

## 19. Failure Scenarios

1. **Planner failure:** If the LLM cannot create a valid plan, fall back to a simple predefined sequence.
2. **Plugin timeout:** Each plugin execution has a timeout. Long-running plugins return partial results.
3. **Memory store outage:** Degraded to no-memory mode (no RAG). The Kernel continues without memory.
4. **LLM rate limiting:** SK retries with exponential backoff. Configure max retries globally.

## 20. Security

- **Azure AD authentication:** Native support for managed identities and Azure AD.
- **Filter-based redaction:** Use `IPromptFilter` to redact sensitive data from prompts before sending to the LLM.
- **Plugin permission model:** Plugins run in the Kernel's process. Scope plugin access with .NET CAS or Python sandboxing.
- **Function approval gates:** `IFunctionFilter` can block invocation of sensitive functions.
- **Telemetry data governance:** Semantic Kernel's telemetry hooks ensure no user data leaks to third parties.

## 21. Monitoring

- **OpenTelemetry:** SK emits OpenTelemetry traces for kernel invocations, plugin calls, and AI requests.
- **Azure Application Insights:** First-class integration. All events are correlated with a single operation ID.
- **Metrics:** Track plugin latency, LLM token usage, planner success rate, filter execution time.
- **Logging:** Built-in `ILogger<T>` integration. Structured JSON logs for every kernel operation.

## 22. Interview Questions

1. Explain the difference between semantic functions and native functions in SK.
2. How does the Planner work? What are its limitations?
3. Describe the filter pipeline and how you'd implement a profanity filter.
4. How does Semantic Kernel's memory system work?
5. Compare SK's plugin model with LangChain's tool model.
6. How would you integrate SK with an existing .NET enterprise application?
7. Explain the role of the Kernel object in orchestration.
8. How does SK handle function calling with Azure OpenAI?

## 23. Cheat Sheet

| Task | C# | Python |
|------|----|--------|
| Create kernel | `var kernel = Kernel.CreateBuilder().Build();` | `kernel = Kernel()` |
| Add AI service | `.AddAzureOpenAIChatCompletion(...)` | `kernel.add_service(AzureChatCompletion(...))` |
| Import plugin | `kernel.ImportPluginFromDirectory("Plugins")` | `kernel.add_plugin(MyPlugin(), "name")` |
| Invoke function | `await kernel.InvokeAsync(plugin["func"])` | `await kernel.invoke(plugin["func"])` |
| Invoke prompt | `await kernel.InvokePromptAsync("Hello")` | `await kernel.invoke_prompt("Hello")` |
| Memory save | `await kernel.Memory.SaveInformationAsync(...)` | `kernel.memory.save_information(...)` |
| Memory search | `await kernel.Memory.SearchAsync(...)` | `kernel.memory.search(...)` |

## History

Microsoft announced Semantic Kernel in March 2023 as part of its AI Copilot strategy. The framework was designed from the ground up to integrate with the Microsoft ecosystem — Azure OpenAI, Microsoft Graph, and the Copilot stack. Version 1.0 GA was released in late 2023 for .NET. Python and Java SDKs followed in 2024. Semantic Kernel is the official SDK for building Microsoft Copilot extensions and is used internally at Microsoft for various AI orchestration workloads. The framework has over 20k GitHub stars and an active, Microsoft-backed development team.

## Core Components Explained

- **Kernel:** Central orchestrator. Holds AI services, plugins, memory, and filters. All operations go through the kernel.
- **Plugins:** Collections of functions. Can be native (code) or semantic (prompts). Functions are discovered dynamically by the planner.
- **Planner:** Auto-generates a sequence of function calls to fulfill a goal. Uses the LLM to select and order functions. Not guaranteed to produce correct plans.
- **Memory:** Vector storage for semantic memory. `SaveInformationAsync` stores text, `SearchAsync` retrieves by similarity.
- **Filters:** Middleware hooks that wrap function invocation and prompt rendering. Enable guardrails, logging, telemetry, and security policies.

## Installation

```bash
# Python
pip install semantic-kernel
# .NET
dotnet add package Microsoft.SemanticKernel
# Java
<dependency>
  <groupId>com.microsoft.semantickernel</groupId>
  <artifactId>semantickernel-api</artifactId>
</dependency>
```

## Advanced Features

- **OpenAPI plugins:** Import any OpenAPI specification as a plugin — SK generates function definitions from the spec.
- **Multi-modal:** Support for image inputs in Azure OpenAI GPT-4 Vision.
- **Grounding:** Integration with Bing Search API for grounding LLM responses in real-world data.
- **Stepwise planner:** Iterative planner that re-plans after each step (more reliable for complex tasks).
- **Handlebars planner:** Uses Handlebars templates for flexible plan generation.
- **Chat history:** Built-in `ChatHistory` object for managing conversation context.
- **Auto function calling:** SK automatically calls functions based on the LLM's `function_call` response.

## Performance Tuning
1. **Reduce planner overhead:** Cache plan results for identical requests. Use the cached plan ID as a lookup key.
2. **Batch memory operations:** `SaveInformationAsync` supports batch mode for high-throughput ingestion.
3. **Streaming for UX:** Use `streaming: true` on chat completions. Pipeline streaming events directly to the UI.
4. **Plugin cold start:** Warm-up plugins during kernel initialisation by invoking a no-op function.
5. **Azure OpenAI latency:** Use `azureOpenAI` with `use west europe for regional proximity. Enable streaming for first-token latency.

## Debugging Tips
1. `kernel.Plugins` lists all registered plugins and functions.
2. Set `Microsoft.SemanticKernel` to `Information` log level for verbose output.
3. Use `HandlebarsPlanner` with `Verbose = true` to see the generated plan.
4. Check `function.ExecutionSettings` for prompt template configuration.
5. Common errors: `FunctionNotAvailableException` (plugin not imported), `PlanCreationException` (LLM failed to generate plan).

## Production Deployment
1. **Azure Container Apps:** Deploy SK as a containerised API. Use Azure OpenAI with managed identity.
2. **Plugin sandboxing:** Run untrusted plugins in separate AppDomain or container. Filter-based access control for trusted plugins.
3. **Telemetry:** Add `OpenTelemetry` exporter. Use Application Insights for correlation and alerting.
4. **Secrets management:** Use Azure Key Vault for API keys and connection strings. SK supports `IConfiguration` integration.
5. **CI/CD:** Unit tests with mock AI services. Integration tests with real Azure OpenAI in a test environment.
