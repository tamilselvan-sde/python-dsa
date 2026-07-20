# LangChain Framework

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

LangChain is an open-source framework released by LangChain Inc. in October 2022 that simplifies building applications powered by large language models (LLMs). It provides a modular, composable architecture for chaining together LLM calls, tools, data sources, and memory into coherent pipelines. LangChain supports multiple LLM providers (OpenAI, Anthropic, Google, Hugging Face, etc.), vector stores (Pinecone, Weaviate, Chroma, etc.), and tool integrations.

## 2. Why do we need it?

Building production LLM applications requires stitching together disparate components: prompt templates, LLM calls, external APIs, vector databases, memory systems, and evaluation pipelines. Without a framework, developers manually handle token management, retries, parsing, and orchestration. LangChain abstracts these concerns, providing standardised interfaces for chains, agents, retrievers, and document loaders, drastically reducing boilerplate.

## 3. Real-world Example

**GitHub Copilot Chat RAG pipeline:** A code-assistance chatbot that indexes a company's internal documentation into a vector store. When a developer asks "How do I deploy service X?", LangChain retrieves relevant chunks from Pinecone, constructs a prompt with context, calls GPT-4, and returns an answer grounded in company docs. The chain includes memory for follow-up questions and a tool for executing shell commands.

## 4. Architecture Diagram (ASCII)

```
             +-------------------+
             |    User Input     |
             +---------+---------+
                       |
              +--------v--------+
              |   PromptTemplate |
              +--------+--------+
                       |
              +--------v--------+
              |    LLM (Model)   |
              +--------+--------+
                       |
           +-----------+-----------+
           |                       |
    +------v------+         +------v------+
    | OutputParser |         |    Tools    |
    +------+------+         +------+------+
           |                       |
           +-----------+-----------+
                       |
              +--------v--------+
              |     Memory      |
              +--------+--------+
                       |
              +--------v--------+
              |    Response     |
              +-----------------+
```

## 5. Internal Working

LangChain's core abstraction is the **Chain**. A chain wraps a sequence of calls: format prompt → call LLM → parse output → (optionally) call a tool → repeat. Each step is a `Runnable` object with a standard `invoke(input)` interface, allowing arbitrary composition via the pipe operator (`|`). Behind the scenes, LangChain manages token counting, callbacks for logging, retry logic with exponential backoff, and batching of parallel requests.

The **Agent** abstraction extends chains by giving the LLM access to tools. The LLM outputs a structured action (tool name + arguments), which the agent executes, feeds the result back, and loops until a final answer is produced.

## 6. Production Flow

```
User Input → Input Guardrails → Retriever (if RAG) → Prompt Assembly →
LLM Call (with retry + fallback) → Output Parsing → Output Guardrails →
Tool Execution (if agent) → Memory Update → Response → Monitoring
```

Each arrow is a `Runnable` that can be traced via LangSmith (LangChain's observability platform). In production, invocations are routed through a runnable executor that handles concurrency, timeouts, and error propagation.

## 7. HLD

```
                   +--------------------+
                   |   API Gateway      |
                   +--------+-----------+
                            |
              +-------------+-------------+
              |             |             |
        +-----v-----+ +----v----+ +-----v-----+
        | Chat Bot  | | Search  | | Document  |
        | Service   | | Service | | Pipeline  |
        +-----+-----+ +----+----+ +-----+-----+
              |             |             |
        +-----v-------------v-------------v-----+
        |         LangChain Runtime             |
        |  (Chain/Agent Executor + Memory)      |
        +-----+-------------+-------------+-----+
              |             |             |
        +-----v-----+ +----v----+ +-----v-----+
        |  LLM API  | | Vector  | | External  |
        | (OpenAI)  | |  Store  | |   APIs    |
        +-----------+ +---------+ +-----------+
```

## 8. LLD

Key classes and their relationships:

```
Runnable (ABC)
  ├── RunnableSequence (pipe operator chains)
  ├── RunnableParallel (parallel execution)
  ├── RunnableLambda (wrap any Python function)
  ├── ChatOpenAI / ChatAnthropic (LLM wrappers)
  ├── PromptTemplate / ChatPromptTemplate
  ├── StrOutputParser / PydanticOutputParser
  └── RunnablePassthrough (pass input through)

BaseChatMemory
  ├── ConversationBufferMemory
  ├── ConversationSummaryMemory
  └── VectorStoreRetrieverMemory

BaseRetriever
  ├── VectorStoreRetriever
  ├── MultiQueryRetriever
  └── EnsembleRetriever

AgentExecutor
  ├── OpenAIFunctionsAgent
  ├── XMLAgent
  └── StructuredChatAgent
```

## 9. Python Implementation

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# RAG chain
vectorstore = Chroma(persist_directory="./chroma_db",
                     embedding_function=OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

template = """Answer based on context:
Context: {context}
Question: {question}"""
prompt = ChatPromptTemplate.from_template(template)
model = ChatOpenAI(model="gpt-4", temperature=0)

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

result = rag_chain.invoke("How do I deploy service X?")
print(result)
```

## 10. Folder Structure

```
project/
├── chains/
│   ├── __init__.py
│   ├── rag_chain.py
│   └── agent_chain.py
├── prompts/
│   ├── templates.py
│   └── examples.py
├── tools/
│   ├── __init__.py
│   ├── search_tool.py
│   └── api_tool.py
├── memory/
│   ├── __init__.py
│   └── conversation_memory.py
├── retrievers/
│   ├── __init__.py
│   └── custom_retriever.py
├── app.py
├── config.yaml
└── requirements.txt
```

## 11. Configuration

```yaml
# config.yaml
llm:
  provider: openai
  model: gpt-4
  temperature: 0
  max_tokens: 2048

vectorstore:
  type: chroma
  persist_directory: ./chroma_db
  embedding_model: text-embedding-3-small

retriever:
  k: 4
  score_threshold: 0.7

chain:
  verbose: true
  max_retries: 3
  timeout: 30
```

## 12. Flowchart

```
        Start
          |
          v
    Receive Input
          |
          v
    Check Cache (LangChain Cache)
          |
    +-----+-----+
    | Hit | Miss |
    +-----+-----+
          |     |
        Return  v
          |   Build Prompt
          |     |
          |     v
          |  Call LLM
          |     |
          |     v
          |  Parse Output
          |     |
          |     v
          |  [Agent?] ---> Tool Call Loop
          |     |
          |     v
          |  Update Memory
          |     |
          |     v
          |  Cache Result
          |     |
          +---> Return
```

## 13. Sequence Diagram

```
User         Chain         LLM         VectorDB       Memory
 |            |             |             |             |
 |--ask()---->|             |             |             |
 |            |--retrieve-->|             |             |
 |            |             |--embed----->|             |
 |            |             |<--chunks----|             |
 |            |--build------|             |             |
 |            |  prompt      |             |             |
 |            |--invoke----->|             |             |
 |            |<--response---|             |             |
 |            |--parse------>|             |             |
 |            |--save------->|             |             |
 |            |              |             |            |
 |<--answer---|              |             |            |
```

## 14. Pros

- Huge ecosystem of integrations (500+)
- Active community and frequent releases
- LangSmith integration for tracing and debugging
- Modular and composable design
- Supports nearly every model provider and vector store
- Extensive documentation and examples

## 15. Cons

- Rapid API churn — code breaks between minor versions
- Over-abstraction leads to a steep learning curve
- Debugging chain internals is difficult without LangSmith
- Heavy dependency footprint
- Hard to customise when you deviate from happy path

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LlamaIndex | Data-centric RAG and indexing pipelines |
| Haystack | NLP-focused production pipelines |
| Semantic Kernel | .NET/enterprise Microsoft ecosystem |
| Pydantic AI | Type-safe, Pydantic-first agent design |
| Direct LLM API | Simple one-off calls, no orchestration needed |

## 17. Performance Considerations

- **Token usage:** Use `with_message_history` to avoid re-sending entire conversation history on every turn. Implement `BaseCache` (e.g., Redis or SQLite) to cache LLM responses for identical inputs.
- **Latency:** `RunnableParallel` runs independent branches concurrently. Use `astream()` for streaming token-by-token responses to the user. Batch embeddings in chunks of 50–100 texts.
- **Memory:** `ConversationSummaryMemory` reduces token overhead by summarising history. For long sessions, use `VectorStoreRetrieverMemory` to selectively recall relevant past exchanges.
- **Retries:** Configure `max_retries` and `timeout` on the LLM wrapper. Implement fallback LLMs via `RunnableBranch`.

## 18. Scaling to Millions

1. **Stateless chain workers:** Horizontal scaling with Kubernetes — each pod runs the same chains. Use a message queue (RabbitMQ / Kafka) to dispatch requests.
2. **Shared memory backend:** Replace in-memory memory with Redis-backed `RedisChatMessageHistory` or PostgreSQL-backed history.
3. **Vector store sharding:** Partition the vector index across nodes (Pinecone pods, Qdrant shards). Use `EnsembleRetriever` to merge results.
4. **Caching layer:** Distributed cache via Redis or Memcached for both LLM responses and retrieved documents. Cache invalidation uses TTL + versioned model IDs.
5. **Rate limiting:** Token-bucket rate limiter per user/API key. Implement queue-based load levelling with priority queues for premium users.
6. **Batching:** LangChain's `batch()` method sends multiple prompts in a single API call (OpenAI supports batching at 50% cost).

## 19. Failure Scenarios

1. **LLM API outage:** Configure fallback models (`model=ChatAnthropic()` as secondary). Circuit breaker pattern via `tenacity` retries with exponential backoff.
2. **Vector store down:** Degraded mode — skip retrieval and use only LLM knowledge. Health-check the vector store before each query.
3. **Token limit exceeded:** Pre-truncate conversation history using a sliding window. Use `token_counter` to estimate before sending.
4. **Tool timeout:** Set per-tool timeouts. Failed tools return an error message to the LLM, which can retry or apologise.
5. **Prompt injection:** Input guardrails via `GuardrailsOutputParser`. Never pass raw user input into tool arguments without validation.

## 20. Security

- **Prompt injection prevention:** Validate user input with `NeMo Guardrails` integration. Sanitise tool call arguments. Never include system prompts in returned output.
- **API key management:** Use environment variables or a secrets manager (HashiCorp Vault, AWS Secrets Manager). LangChain's `.env` loader reads from `.env` file.
- **Tool sandboxing:** Execute tools (e.g., Python REPL, shell) in isolated Docker containers. Restrict filesystem and network access.
- **PII redaction:** `PresidioPII` integration automatically redacts PII from traces and logs before they reach LangSmith.

## 21. Monitoring

- **LangSmith:** Traces every runnable invocation, records latency, token usage, and model responses. Supports feedback loops for RLHF.
- **Metrics to track:** LLM latency (p50/p95/p99), token usage per user, cache hit rate, retry rate, tool success rate, vector store query time.
- **Logging:** Structured JSON logs emitted per chain step. Integrate with ELK stack or DataDog. Include trace IDs for end-to-end correlation.
- **Alerts:** PagerDuty alert when error rate exceeds 1% or p95 latency exceeds 10s. LangSmith webhooks trigger on run failures.

## 22. Interview Questions

1. How does LangChain's `Runnable` protocol differ from the legacy `Chain` API?
2. Explain how `RunnableParallel` enables concurrent execution and what edge cases it creates.
3. How would you implement a custom memory system that summarises conversations selectively?
4. Describe the agent decision loop — how does `AgentExecutor` decide when to stop?
5. How does LangChain handle token counting across different model providers?
6. Implement a self-corrective RAG pipeline using `RunnableBranch`.
7. How would you scale LangChain agents to handle 10,000 concurrent users?
8. Explain the difference between `tool` and `toolkit` in LangChain.

## 23. Cheat Sheet

| Concept | Key Class | Usage |
|---------|-----------|-------|
| Chain composition | `RunnableSequence` | `chain = prompt \| llm \| parser` |
| Parallel execution | `RunnableParallel` | `chain = {"a": chain1, "b": chain2}` |
| Conditional routing | `RunnableBranch` | `branch = RunnableBranch((cond, chain), default)` |
| Agent creation | `create_openai_functions_agent` | `agent = create_openai_functions_agent(llm, tools, prompt)` |
| Memory | `ConversationBufferMemory` | `memory.chat_memory.add_user_message("hi")` |
| Caching | `InMemoryCache` | `llm.cache = InMemoryCache()` |
| Streaming | `chain.stream(input)` | Yields tokens incrementally |

## History

LangChain began as a personal project by Harrison Chase in October 2022. The first public commit was a simple chain abstraction. By early 2023, it had gained explosive adoption, reaching 50k+ GitHub stars. Version 0.1 was released in January 2024 with the `Runnable` protocol. LangChain Inc. raised $25M Series A (led by Sequoia) and later acquired the company behind Zapier's natural language API. The `langchain-core` package was split from integrations in v0.2. As of 2025, LangChain is the most widely adopted LLM framework with over 500 integrations and LangSmith as its commercial observability platform.

## Core Components Explained

- **Runnable:** The universal interface — any component with `invoke()`, `batch()`, `stream()`, and `astream()`. Runnables compose via `|`, `pipe()`, and `RunnableParallel`.
- **Chat Models:** Wrappers for LLM chat APIs (`ChatOpenAI`, `ChatAnthropic`). Handle tokenisation, retries, and streaming natively.
- **Prompt Templates:** `ChatPromptTemplate`, `MessagesPlaceholder`, `FewShotPromptTemplate` for dynamic prompt construction.
- **Output Parsers:** `StrOutputParser` (raw text), `PydanticOutputParser` (structured), `CommaSeparatedListOutputParser`.
- **Memory:** `ConversationBufferMemory`, `ConversationSummaryMemory`, `VectorStoreRetrieverMemory`, `PostgresChatMessageHistory`.
- **Retrievers:** `VectorStoreRetriever`, `MultiQueryRetriever` (generates multiple queries), `ContextualCompressionRetriever` (reranks).
- **Tools:** `Tool` dataclass with name, description, and function. Built-in tools for `wikipedia`, `duckduckgo`, `python_repl`, and more.
- **Callbacks:** `BaseCallbackHandler` for logging, tracing, streaming to UI. LangSmith callback handler for observability.

## Installation

```bash
pip install langchain langchain-openai langchain-community langchainhub
pip install chromadb tiktoken
# or with langchain-cli
pip install langchain-cli
langchain app new my-app
```

## Advanced Features

- **LangGraph integration:** Build stateful, cyclic, and multi-agent workflows on top of LangChain runnables. Enables persistence, branching, human-in-the-loop, and parallel agent execution.
- **LangServe:** Deploy LangChain chains as production REST APIs with automatic streaming support. Generated OpenAPI docs with `/invoke` and `/stream` endpoints. Built-in rate limiting and request validation.
- **LangSmith:** Full tracing, evaluation datasets, and online feedback. Compare prompt versions side-by-side. Annotate runs for fine-tuning data.
- **Custom runnables:** `RunnableGenerator` for complex streaming logic. `RunnableLambda` for wrapping arbitrary Python functions. `@chain` decorator for simple fallback chains.
- **Async support:** Native `async`/`await` for all I/O-bound operations. `asyncio.gather()` for parallel tool calls.
- **Streaming events:** `astream_events()` yields granular events (on_chain_start, on_chat_model_stream, on_tool_end) for building responsive UIs.

## Performance Tuning

1. **Batching:** Use `chain.batch([input1, input2, ...])` instead of multiple `invoke()` calls — LangChain sends batched LLM requests.
2. **Caching:** Enable `InMemoryCache` or `RedisCache` for LLM responses. Set TTL based on staleness tolerance.
3. **Embedding caching:** Cache document embeddings in memory to avoid recomputing on every query.
4. **Streaming:** Use `chain.stream()` for token-by-token responses — lowers perceived latency even if total time is the same.
5. **Parallel retrieval:** Use `RunnableParallel` to query multiple retrievers simultaneously and merge results.
6. **Prompt compression:** Use `LLMChainExtractor` to compress retrieved documents before sending to the LLM.

## Debugging Tips

1. Set `langchain.debug = True` for verbose logging of every step.
2. Use `chain.get_graph().print_ascii()` to visualise the chain topology.
3. LangSmith traces capture full I/O for every runnable — inspect token counts, latency, and model outputs.
4. Use `chain.with_config({"run_name": "my-debug-chain"})` for easier identification in logs.
5. Common errors: `OutputParserException` (LLM returned malformed JSON), `IndexError` (empty retriever results).

## Production Deployment

1. **Containerisation:** Use Docker with `gunicorn` + `uvicorn` workers. LangServe auto-generates FastAPI app.
2. **Scaling:** Horizontal scaling with Kubernetes. Stateless chains scale effortlessly. Stateful agents use Redis-backed memory.
3. **Secrets:** Never hardcode API keys. Use environment variables or a secrets manager. LangChain's `load_dotenv()` reads from `.env`.
4. **Rate limiting:** LangServe supports built-in rate limiting. For custom deployments, use middleware or API gateway (Kong, Envoy).
5. **Monitoring:** LangSmith for traces. Prometheus + Grafana for system metrics (latency, throughput, error rate).
6. **CI/CD:** Run LangChain unit tests + integration tests (with mock LLM endpoints) in CI. Use LangSmith datasets for regression testing.
