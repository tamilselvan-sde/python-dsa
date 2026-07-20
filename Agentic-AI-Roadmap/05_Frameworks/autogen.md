# Microsoft AutoGen

## 1. What is it?

AutoGen is an open-source multi-agent conversation framework developed by Microsoft Research (released August 2023). It enables building LLM applications through **conversational multi-agent coordination** — agents converse with each other to solve tasks, with support for human-in-the-loop, code execution, and tool use. The core idea is that complex tasks are solved not by a single agent, but by multiple specialised agents that talk to each other.

## 2. Why do we need it?

Complex tasks require multiple capabilities — reasoning, coding, searching, executing, validating. Monolithic agents struggle with this breadth. AutoGen solves:
- **Conversational orchestration:** Agents discover solutions through dialogue, not fixed pipelines
- **Code generation + execution:** One agent writes code, another runs and debugs it
- **Human-in-the-loop:** Humans can participate in the conversation at any point
- **Flexible agent topologies:** Two-agent, group chat, or nested conversations
- **Enterprise-grade:** Microsoft-backed, used in Azure AI services

## 3. Real-world Example

**Automated data analysis pipeline:**
1. **User** asks "Analyse this CSV and create a visualisation"
2. **AssistantAgent** writes Python code for analysis
3. **UserProxyAgent** executes the code in a sandbox
4. Code fails → **AssistantAgent** reads error, fixes code
5. Code succeeds → **AssistantAgent** interprets results
6. Both agents converse until the user is satisfied

## 4. Architecture Diagram (ASCII)

```
         +-----------------------------------------------+
         |               AutoGen Runtime                 |
         +-----------------------------------------------+
         |                                                   |
    +----v----+    +----v----+    +----v----+    +----v----+
    | Assistant|    | User    |    | Group   |    | Tool    |
    | Agent    |    | Proxy   |    | Chat    |    | Agent   |
    | (LLM)   |    | (Human) |    | Manager |    | (Code)  |
    +---------+    +---------+    +---------+    +---------+
         |              |              |              |
         +------+-------+-------+------+--------------+
                |       |       |      |
           +----v---v---v---v---v------v----+
           |       Conversation Bus         |
           |  (Message passing between      |
           |   agents via 2-way channels)   |
           +--------------------------------+
```

## 5. Internal Working

AutoGen's core is the **Agent** abstract class. Agents communicate by sending and receiving messages via `send()` and `receive()` methods. Key implementations:

1. **ConversableAgent:** Base class. Has a `name`, `llm_config`, `human_input_mode`, and `code_execution_config`. Supports registering reply functions and tool functions.
2. **AssistantAgent:** A ConversableAgent backed by an LLM. It generates replies (text, tool calls, code) without executing code.
3. **UserProxyAgent:** A ConversableAgent that can execute code, take human input, and call tools. It acts as the bridge between the LLM and the real world.

**Two-agent chat:** AssistantAgent proposes solutions → UserProxyAgent executes → feeds results back → loop.

**GroupChat:** Multiple agents chat via a GroupChatManager that orchestrates turn-taking and selects the next speaker.

## 6. Production Flow

```
User Input → UserProxyAgent receives → forwards to AssistantAgent
AssistantAgent generates response (code/tool call)
→ UserProxyAgent executes code / calls tool
→ Result sent back to AssistantAgent
→ If satisfied → final answer to user
→ If not → AssistantAgent refines → loop
→ Human can interject at any time
```

## 7. HLD

```
                   +-----------------------+
                   |   API Gateway         |
                   +----------+------------+
                              |
                   +----------+------------+
                   |   AutoGen Service     |
                   |   (Agent Runtime)    |
                   +----------+------------+
                              |
              +---------------+---------------+
              |               |               |
      +-------v----+   +-----v------+  +-----v------+
      | Assistant  |   | UserProxy  |  | Tool       |
      | Agents     |   | Agents     |  | Executors  |
      +------------+   +------------+  +------------+
              |               |               |
      +-------v----+   +-----v------+         |
      | LLM APIs   |   | Code       |         |
      | (Azure)    |   | Sandbox    |         |
      +------------+   | (Docker)   |         |
                       +------------+         |
                                      +------v------+
                                      | External    |
                                      | APIs        |
                                      +-------------+
```

## 8. LLD

```
ConversableAgent
  ├── name: str
  ├── llm_config: dict | None
  ├── human_input_mode: str  # NEVER, TERMINATE, ALWAYS
  ├── code_execution_config: dict
  ├── register_reply(reply_func)
  ├── register_tool(tool_func)
  ├── send(msg, recipient)
  ├── receive(msg, sender)
  └── generate_reply(messages)

AssistantAgent(ConversableAgent)
  ├── system_message: str
  └── generate_reply()  # uses LLM

UserProxyAgent(ConversableAgent)
  ├── execute_code()    # runs code in sandbox
  └── get_human_input() # prompts user

GroupChat
  ├── agents: List[ConversableAgent]
  ├── messages: List[dict]
  └── select_speaker()  # decides next speaker

GroupChatManager
  ├── groupchat: GroupChat
  └── run_chat()        # orchestrates group conversation
```

## 9. Python Implementation

```python
import autogen
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

# Configuration
llm_config = {
    "config_list": [{"model": "gpt-4", "api_key": "sk-..."}],
    "temperature": 0,
    "timeout": 120,
}

# Assistant agent (LLM-powered)
assistant = AssistantAgent(
    name="CodingAssistant",
    llm_config=llm_config,
    system_message="You are a Python expert. Write clear, well-documented code."
)

# User proxy agent (executes code, takes human input)
user_proxy = UserProxyAgent(
    name="UserProxy",
    human_input_mode="TERMINATE",  # asks human only at termination
    code_execution_config={
        "work_dir": "coding",
        "use_docker": True,  # run in Docker container
        "last_n_messages": 3,
    },
    llm_config=False,  # no LLM, only execution
)

# Initiate chat
user_proxy.initiate_chat(
    assistant,
    message="Create a Python script that downloads S&P 500 data "
            "and plots a moving average crossover strategy."
)

# For streaming output
user_proxy.initiate_chat(
    assistant,
    message="Analyse this CSV file",
    max_turns=10,
)
```

### Group Chat Example

```python
# Multiple agents in group chat
researcher = AssistantAgent(
    name="Researcher",
    system_message="You are a research expert. Find facts and data.",
    llm_config=llm_config,
)

analyst = AssistantAgent(
    name="Analyst",
    system_message="You are a data analyst. Analyse data and draw conclusions.",
    llm_config=llm_config,
)

writer = AssistantAgent(
    name="Writer",
    system_message="You are a technical writer. Create compelling reports.",
    llm_config=llm_config,
)

groupchat = GroupChat(
    agents=[researcher, analyst, writer, user_proxy],
    messages=[],
    max_round=20,
    speaker_selection_method="auto",  # or "round_robin" or "random"
)

manager = GroupChatManager(
    groupchat=groupchat,
    llm_config=llm_config,
)

user_proxy.initiate_chat(
    manager,
    message="Research AI agent frameworks and write a comparison report."
)
```

## 10. Folder Structure

```
autogen-app/
├── agents/
│   ├── __init__.py
│   ├── custom_agents.py
│   └── tool_agents.py
├── tools/
│   ├── __init__.py
│   ├── search_tool.py
│   └── api_tool.py
├── workflows/
│   ├── __init__.py
│   ├── two_agent_chat.py
│   └── group_chat.py
├── docker/                # Docker sandbox config
│   └── Dockerfile
├── coding/                # Code execution workdir
├── api/
│   └── main.py
├── tests/
├── config.yaml
└── pyproject.toml
```

## 11. Configuration

```yaml
# config.yaml
llm:
  config_list:
    - model: gpt-4
      api_key: ${OPENAI_API_KEY}
      base_url: https://api.openai.com/v1
  temperature: 0
  timeout: 120

code_execution:
  work_dir: ./coding
  use_docker: true
  timeout: 60
  last_n_messages: 3

agent:
  human_input_mode: TERMINATE
  max_consecutive_auto_reply: 10
```

## 12. Flowchart

```
User Request
     |
     v
UserProxyAgent receives message
     |
     v
Forwards to AssistantAgent
     |
     v
AssistantAgent generates response
(LLM call: text or code or tool call)
     |
     v
UserProxyAgent receives response
     |
     +-- If code block: execute in sandbox
     |       |
     |       +-- Success: return output
     |       +-- Error: return error message
     |
     +-- If tool call: execute tool
     |       |
     |       +-- Return tool result
     |
     +-- If text: pass through
     |
     v
Send execution result back to AssistantAgent
     |
     v
AssistantAgent checks if task is complete
     |
     +-- Yes: send "TERMINATE" message
     |        UserProxyAgent asks human
     |        Human approves or asks for changes
     |
     +-- No: generate next iteration
              (fix errors, refine, expand)
              |
              v
              Loop back to UserProxyAgent execution
```

## 13. Sequence Diagram

```
User       UserProxy      Assistant     Docker Code    LLM
 |            |              |              |           |
 |--ask()---->|              |              |           |
 |            |--send()----->|              |           |
 |            |              |--LLM call--------------->|
 |            |              |<--response--------------|
 |            |<--response---|              |           |
 |            |--exec code-->|              |           |
 |            |              |--run-------->|           |
 |            |              |<--output-----|           |
 |            |<--result-----|              |           |
 |            |--send()----->|              |           |
 |            |              |--LLM call--------------->|
 |            |              |<--response--------------|
 |            |<--response---|              |           |
 |            |--TERMINATE-->|              |           |
 |<--answer---|              |              |           |
```

## 14. Pros

- **Conversational approach** — intuitive, agents discover solutions through dialogue
- **Strong code execution** — Docker sandboxed, supports multiple languages
- **Flexible agent topologies** — 2-agent, group chat, nested, custom
- **Human-in-the-loop** — fine-grained control over agent autonomy
- **Microsoft-backed** — production trust, Azure integration, ongoing research
- **Open-source** — Apache 2.0 license, active development

## 15. Cons

- **Conversation overhead** — agents talk a lot, consuming tokens for coordination
- **Complexity** — many concepts (agents, group chats, nested chats, register replies)
- **Code execution risks** — Docker sandbox required for security, adds setup overhead
- **Debugging difficulty** — tracking multi-agent conversations is challenging
- **Documentation** — evolving quickly, examples may lag behind API changes

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangGraph | Stateful graph orchestration, not conversational |
| CrewAI | Role-based, simpler multi-agent model |
| Semantic Kernel | .NET ecosystem, Microsoft stack |
| OpenAI Assistants | Managed, simpler alternative |
| AutoGen.NET | C# version for .NET developers |

## 17. Performance Considerations

- **Token consumption:** Each conversation round consumes tokens for the full message history. Use `max_consecutive_auto_reply` and `max_turns` to limit.
- **Code execution latency:** Sandboxed code execution adds 1-3s startup. For high throughput, keep Docker containers warm.
- **LLM latency:** Group chats with many agents sequentially call the LLM for each speaker selection. Use `speaker_selection_method="round_robin"` to skip LLM-based selection.
- **Memory:** AutoGen does not have built-in vector memory. For long-running conversations, implement external memory with summarization.
- **Parallelism:** AutoGen's conversation model is inherently sequential. For parallel tool calls, use nested chats.

## 18. Scaling to Millions

1. **Stateless agent definitions:** Agent configurations are static. Scale horizontally by running multiple agent sessions in Kubernetes pods.
2. **Code execution pool:** Pre-warmed Docker containers in a pool. AutoGen checks out a container for execution, returns it to pool after.
3. **Message persistence:** Store conversation history in PostgreSQL or Cosmos DB. AutoGen can load history for long-running sessions.
4. **Nested chats:** For complex tasks, use nested chats that spawn sub-conversations in parallel. GroupChatManager routes messages.
5. **Azure integration:** Use Azure OpenAI for enterprise-scale LLM access with built-in rate limiting and content filtering.
6. **Caching:** LLM response caching via `redis`. Identical queries with the same context are served from cache.

## 19. Failure Scenarios

1. **Infinite conversation loop:** `max_consecutive_auto_reply` default is 10. Set lower for critical paths.
2. **Code execution timeout:** Docker execution has `timeout` config. Unresponsive code gets killed.
3. **Malicious code execution:** Use Docker sandbox with no network, limited filesystem, and memory limits. Never run code on the host.
4. **LLM rate limit:** AutoGen does not have built-in rate limiting. Implement via `config_list` with multiple API keys that rotate.
5. **Message format errors:** Agent expects specific message format. Serialisation errors in custom agents can crash the conversation.

## 20. Security

- **Code execution isolation:** Always use `use_docker=True` in production. The Docker container should have no network access and a read-only filesystem except the working directory.
- **API key management:** Use `config_list` with environment variables. Never hardcode keys. Azure Key Vault integration recommended.
- **Human approval gates:** For destructive operations (delete, deploy, transfer), set `human_input_mode="ALWAYS"`.
- **Prompt injection:** Sanitise external tool outputs before feeding to the LLM. Validate tool call arguments.
- **Audit logging:** Log all messages between agents. AutoGen's `register_reply` can be used for custom logging.

## 21. Monitoring

- **Conversation tracing:** Log all messages with timestamps, agent name, token count, and duration.
- **Metrics:** Track conversations started, completed, terminated (by human vs. agent), average turns, average tokens consumed.
- **Alerts:** Alert on abnormal token consumption (>5x average), excessive turns, frequent code execution failures.
- **AutoGen + LangSmith:** AutoGen messages can be forwarded to LangSmith for visual tracing.

## 22. Interview Questions

1. Explain the difference between AssistantAgent and UserProxyAgent.
2. How does GroupChat speaker selection work? What methods are available?
3. Describe how AutoGen's code execution is sandboxed and why this matters.
4. How would you implement a human-in-the-loop approval gate for a financial transaction agent?
5. Compare AutoGen's conversational approach to LangGraph's state-graph approach.
6. What is a nested chat and when would you use one?
7. How does AutoGen handle context windows for long-running conversations?
8. Design a multi-agent system for automated customer support that can escalate to humans.

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Create assistant | `AssistantAgent(name="A", llm_config=llm_config)` |
| Create user proxy | `UserProxyAgent(name="U", code_execution_config={...})` |
| Start 2-agent chat | `user_proxy.initiate_chat(assistant, message="...")` |
| Group chat | `GroupChat(agents=[a,b,c], messages=[])` |
| Group manager | `GroupChatManager(groupchat=gc, llm_config=llm_config)` |
| Register tool | `assistant.register_for_llm(tool_func, name="my_tool")` |
| Human input mode | `"NEVER"`, `"TERMINATE"`, `"ALWAYS"` |
| Send message | `agent.send("hello", recipient)` |

## History

AutoGen was released by Microsoft Research in August 2023. The paper "AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation" was published in October 2023. Version 0.1 supported basic 2-agent and group chat. Version 0.2 added nested chats, improved code execution, and Azure AI integration. AutoGen became part of Microsoft's AI platform strategy, integrated with Azure Machine Learning and Azure AI Studio. The framework has over 25k GitHub stars and is used internally at Microsoft for various AI orchestration tasks. AutoGen 0.3+ simplified the API and dropped the framework dependency on specific LLM providers.

## Core Components Explained

- **ConversableAgent:** Base class with message sending/receiving, reply registration, and tool registration. All agents extend this.
- **AssistantAgent:** LLM-powered agent that generates replies. It cannot execute code or call tools — it proposes them.
- **UserProxyAgent:** Executes code, calls tools, takes human input. It is the bridge between the LLM and the real world. In fully autonomous mode, `human_input_mode="NEVER"`.
- **GroupChat:** Manages a conversation among multiple agents. Handles message ordering, history, and speaker selection.
- **GroupChatManager:** Orchestrates GroupChat execution. Can be LLM-driven (smart speaker selection) or rule-based (round-robin).
- **Tool registration:** Functions can be registered for the LLM (description + schema) and for execution (actual implementation) via `register_for_llm` and `register_for_execution`.

## Installation

```bash
pip install pyautogen
# With Docker support
pip install pyautogen[docker]
# With optional dependencies
pip install pyautogen[blendsearch,retrievechat,mathchat]
# API key
export OPENAI_API_KEY=sk-...
```

## Advanced Features

- **Nested chats:** Spawn sub-conversations from within a chat. E.g., a customer support agent spawns a separate chat with a billing specialist.
- **RetrieveChat:** Built-in RAG pattern. Agents can query a vector store during conversation.
- **Teachability:** Agents can learn from user feedback and store learnings for future conversations.
- **Async mode:** Support for `async` agent execution.
- **Custom reply functions:** `register_reply(func)` for custom response logic without an LLM.
- **Flexible config_list:** Rotate API keys, fallback models, rate-limit multiple endpoints.

## Performance Tuning

1. **Reduce token waste:** Use `max_turns` to cap conversation length. Use `summary_method="last_msg"` for cheaper summarization.
2. **Code execution warm-up:** Keep Docker container running between conversations. Set `use_docker="container_name"` to reuse.
3. **Group chat optimization:** For simple turn-taking, use `speaker_selection_method="round_robin"` instead of "auto" (which requires an LLM call).
4. **Parallel tool calls:** Use nested chats to execute independent tool calls concurrently.
5. **Cache LLM responses:** AutoGen supports `cache_seed` for deterministic LLM responses during development. In production, use Redis.

## Debugging Tips

1. Enable `autogen.ChatCompletion.start_logging()` to capture all LLM calls.
2. Set `llm_config={"config_list": [...], "verbose": True}` to print raw LLM responses.
3. Use `print(agent.chat_messages)` to inspect the full conversation history.
4. For group chat, inspect `groupchat.messages` for all messages.
5. Common errors: `ValueError: No available config` (API key not set), `TimeoutError` (code execution timeout).

## Production Deployment

1. **Containerise:** Docker container with the AutoGen application. Separate container for code execution sandbox.
2. **Azure deployment:** Use Azure Container Apps for serverless scaling. Azure OpenAI for LLM access with managed rate limiting.
3. **Message persistence:** Cosmos DB or PostgreSQL for conversation history. Enables session recovery on pod restart.
4. **Authentication:** Azure AD authentication for API endpoints. API key validation for external requests.
5. **CI/CD:** Run AutoGen unit tests + integration tests in CI. Use mock LLM endpoints for deterministic testing.
6. **Monitoring:** Application Insights integration for full observability. Custom metrics for agent-level events.
