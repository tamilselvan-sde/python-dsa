# 🧠 Agentic AI Roadmap — From Beginner to Production Architect

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

> **Production-Grade Learning Path for LLMs, RAG, Agents, and AI Systems**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## 📚 What is this?

A **complete production-grade curriculum** (~150 guides) that takes you from **LLM beginner → Senior AI Engineer → Production Architect**. Built for engineers who want to build real-world AI systems, not toy demos.

### Who is this for?
| Role | Focus |
|------|-------|
| **Software Engineers** | Move into ML/AI engineering |
| **ML Engineers** | Learn production deployment & scaling |
| **DevOps/MLOps** | Understand AI system architecture |
| **FAANG Candidates** | Ace system design & ML interviews |
| **Founders/CTOs** | Design scalable AI products |

---

## 🗺️ 16-Week Roadmap

### Month 1: Foundation (Weeks 1-4)

| Week | Topics | Projects | Interview Prep |
|------|--------|----------|----------------|
| **1** | AI vs ML vs DL, Transformers, Attention | Implement attention from scratch (NumPy) | 10 beginner questions |
| **2** | Self-attention, Positional Encoding, RoPE | Build a mini-transformer forward pass | 15 beginner questions |
| **3** | MoE, KV Cache, Flash Attention | Profile KV cache memory savings | 15 beginner questions |
| **4** | Speculative Decoding | Implement draft-target verification | 10 intermediate questions |

**Milestone:** Implement a complete transformer inference pipeline in PyTorch.

**Revision Checklist:**
- [ ] Can explain attention Q/K/V to a 5-year-old
- [ ] Can draw transformer architecture from memory
- [ ] Can calculate KV cache size for a given model
- [ ] Understands why Flash Attention is faster

---

### Month 2: Inference & Retrieval (Weeks 5-8)

| Week | Topics | Projects | Interview Prep |
|------|--------|----------|----------------|
| **5** | vLLM, SGLang, Ollama, HF Transformers | Run a model with vLLM, compare throughput | 10 intermediate questions |
| **6** | Tensor/Pipeline Parallel, Continuous Batching | Deploy with tensor parallelism (2 GPUs) | 10 intermediate questions |
| **7** | RAG, Hybrid Search, Dense/Sparse Retrieval | Build a RAG pipeline with Qdrant + BM25 | 10 intermediate questions |
| **8** | Chunking, Reranking, Semantic Cache | Implement semantic caching with Redis | 10 intermediate questions |

**Milestone:** Deploy a production RAG system with FastAPI + vLLM + Qdrant.

**Revision Checklist:**
- [ ] Can explain continuous batching vs static batching
- [ ] Has deployed a model with tensor parallelism
- [ ] Can design a hybrid search pipeline
- [ ] Understands chunking strategies and their trade-offs

---

### Month 3: Agents & Frameworks (Weeks 9-12)

| Week | Topics | Projects | Interview Prep |
|------|--------|----------|----------------|
| **9** | Agents, Planning, ReAct, Tool Calling | Build a ReAct agent with function calling | 10 senior questions |
| **10** | Memory, Reflection, Self-Correction | Add memory and reflection to agent | 10 senior questions |
| **11** | Multi-Agent, Workflow Agents | Build a multi-agent system (LangGraph) | 10 senior questions |
| **12** | LangChain, LangGraph, LlamaIndex, CrewAI | Implement Agentic RAG with LangGraph | 10 senior questions |

**Milestone:** Build a multi-agent research system with LangGraph.

**Revision Checklist:**
- [ ] Can explain ReAct loop step-by-step
- [ ] Understands agent memory types (STM, LTM, episodic)
- [ ] Has built a multi-agent orchestration system
- [ ] Can compare LangGraph vs CrewAI vs AutoGen

---

### Month 4: Production (Weeks 13-16)

| Week | Topics | Projects | Interview Prep |
|------|--------|----------|----------------|
| **13** | System Design: Chatbot, RAG, Multi-Agent | Design an enterprise chatbot (HLD + LLD) | 10 architecture questions |
| **14** | Docker, K8s, GPU Scheduling, Autoscaling | Containerize and deploy on K8s with GPU | 10 architecture questions |
| **15** | MLOps: MLflow, DVC, Airflow, CI/CD | Set up ML pipeline with CI/CD + registry | 10 troubleshooting scenarios |
| **16** | Monitoring, Observability, Security | Add Prometheus + Grafana + Langfuse | Final mock interview |

**Milestone:** Full production deployment with monitoring, autoscaling, and CI/CD.

**Revision Checklist:**
- [ ] Can design a system handling 10M+ daily requests
- [ ] Understands GPU scheduling and autoscaling strategies
- [ ] Has end-to-end CI/CD for ML pipelines
- [ ] Can monitor TTFT, TPS, GPU utilization, cache hit ratio

---

### Capstone Projects

| Project | Technologies | Difficulty |
|---------|-------------|------------|
| **Enterprise Chatbot** | FastAPI + LangGraph + Qdrant + Redis + K8s | ⭐⭐⭐ |
| **Multi-Agent Research System** | LangGraph + vLLM + ChromaDB + Docker | ⭐⭐⭐⭐ |
| **Real-time Document Search** | FastAPI + Qdrant + BM25 + Redis Cache | ⭐⭐⭐ |
| **AI Code Assistant** | FastAPI + vLLM + Sandbox + WebSocket | ⭐⭐⭐⭐ |
| **Healthcare RAG** | FastAPI + LangChain + Qdrant + FHIR + HIPAA | ⭐⭐⭐⭐⭐ |

---

## 📂 Repository Structure

```
Agentic-AI-Roadmap/
│
├── 01_Foundation/          # Core AI concepts (10 files)
│   ├── ai_vs_ml_vs_dl.md       ─── Foundation distinctions
│   ├── transformers.md          ─── Transformer architecture
│   ├── attention.md             ─── Attention mechanism
│   ├── self_attention.md        ─── Self-attention & MHA
│   ├── positional_encoding.md   ─── Sinusoidal PE
│   ├── rope.md                  ─── Rotary Position Embedding
│   ├── moe.md                   ─── Mixture of Experts
│   ├── kv_cache.md              ─── KV Cache
│   ├── flash_attention.md       ─── Flash Attention
│   └── speculative_decoding.md  ─── Speculative decoding
│
├── 02_LLM_Inference/       # Inference engines & parallelism (10 files)
│   ├── vllm.md                  ─── vLLM engine
│   ├── sglang.md                ─── SGLang
│   ├── ollama.md                ─── Ollama
│   ├── hf_transformers.md       ─── HuggingFace
│   ├── tensor_parallel.md       ─── Tensor parallelism
│   ├── pipeline_parallel.md     ─── Pipeline parallelism
│   ├── continuous_batching.md   ─── Continuous batching
│   ├── paged_attention.md       ─── PagedAttention
│   ├── prefix_cache.md          ─── Prefix caching
│   └── kv_cache_offloading.md   ─── KV cache offloading
│
├── 03_RAG/                # Retrieval systems (10 files)
│   ├── rag.md                   ─── RAG architecture
│   ├── hybrid_search.md         ─── Hybrid search
│   ├── dense_retrieval.md       ─── Dense retrieval
│   ├── sparse_retrieval.md      ─── Sparse retrieval
│   ├── bm25.md                  ─── BM25
│   ├── reranking.md             ─── Reranking
│   ├── semantic_cache.md        ─── Semantic cache
│   ├── query_rewriting.md       ─── Query rewriting
│   ├── chunking.md              ─── Chunking strategies
│   └── metadata_filtering.md    ─── Metadata filtering
│
├── 04_Agentic_AI/         # Agent systems (8 files)
│   ├── agent.md                 ─── AI agents
│   ├── planner.md               ─── Planning
│   ├── memory.md                ─── Agent memory
│   ├── tool_calling.md          ─── Tool calling
│   ├── reflection.md            ─── Reflection
│   ├── multi_agent.md           ─── Multi-agent systems
│   ├── workflow_agents.md       ─── Workflow agents
│   └── autonomous_agents.md     ─── Autonomous agents
│
├── 05_Frameworks/         # AI frameworks (10 files)
│   ├── langchain.md             ─── LangChain
│   ├── langgraph.md             ─── LangGraph
│   ├── llamaindex.md            ─── LlamaIndex
│   ├── crewai.md                ─── CrewAI
│   ├── autogen.md               ─── AutoGen
│   ├── haystack.md              ─── Haystack
│   ├── semantic_kernel.md       ─── Semantic Kernel
│   ├── pydantic_ai.md           ─── Pydantic AI
│   ├── agno.md                  ─── Agno
│   └── openai_agents_sdk.md     ─── OpenAI Agents SDK
│
├── 06_Vector_Databases/   # Vector stores (6 files)
│   ├── chromadb.md              ─── ChromaDB
│   ├── qdrant.md                ─── Qdrant
│   ├── milvus.md                ─── Milvus
│   ├── weaviate.md              ─── Weaviate
│   ├── pinecone.md              ─── Pinecone
│   └── faiss.md                 ─── FAISS
│
├── 07_Architecture/       # System architecture (7 files)
│   ├── hld.md                   ─── High-level design
│   ├── lld.md                   ─── Low-level design
│   ├── scalable_rag.md          ─── Scalable RAG
│   ├── scalable_agents.md       ─── Scalable agents
│   ├── microservices.md         ─── Microservices
│   ├── api_gateway.md           ─── API Gateway
│   └── event_driven.md          ─── Event-driven
│
├── 08_Deployment/         # Production deployment (7 files)
│   ├── docker.md                ─── Docker
│   ├── kubernetes.md            ─── Kubernetes
│   ├── gpu_scheduling.md        ─── GPU scheduling
│   ├── autoscaling.md           ─── Autoscaling
│   ├── nginx.md                 ─── Nginx
│   ├── monitoring.md            ─── Monitoring
│   └── observability.md         ─── Observability
│
├── 09_MLOps/              # MLOps tools (5 files)
│   ├── mlflow.md                ─── MLflow
│   ├── dvc.md                   ─── DVC
│   ├── airflow.md               ─── Airflow
│   ├── cicd.md                  ─── CI/CD
│   └── model_registry.md        ─── Model registry
│
├── 10_System_Design/      # System design studies (5 files)
│   ├── chatbot_design.md        ─── Enterprise chatbot
│   ├── enterprise_rag.md        ─── Enterprise RAG
│   ├── multi_agent_system.md    ─── Multi-agent system
│   ├── ai_assistant.md          ─── AI assistant
│   └── document_search.md       ─── Document search
│
└── 11_FAANG_Interview/    # Interview prep (3 files)
    ├── interview_questions.md   ─── 300 Q&A
    ├── architecture_questions.md ─── 50 architecture Q&A
    └── troubleshooting.md       ─── 50 incident scenarios

Total: ~150 files | ~300,000 words
```

---

## 🎯 How to Use This Repository

### Option 1: Follow the 16-week roadmap (above)
Work through the modules week-by-week. Each week has topics, projects, and interview prep.

### Option 2: Go deep on one topic
Each file is self-contained. Jump straight to what you need.

### Option 3: Interview prep mode
1. Start with `11_FAANG_Interview/interview_questions.md` (300 Q&A)
2. Move to `11_FAANG_Interview/architecture_questions.md` (50 system design)
3. Practice with `11_FAANG_Interview/troubleshooting.md` (50 incidents)

### Option 4: Build mode
Pick a capstone project and build it using the guides as reference.

---

## 🏆 Key Learning Outcomes

By the end of this roadmap, you will be able to:

| Skill | Level |
|-------|-------|
| Explain Transformer architecture in detail | 🎯 Architect |
| Deploy LLMs with vLLM + Kubernetes | 🎯 Senior |
| Build production RAG with hybrid search | 🎯 Senior |
| Design multi-agent systems | 🎯 Architect |
| System design for 10M+ daily requests | 🎯 Staff+ |
| Ace FAANG ML/AI interviews | 🎯 Ready |

---

## 🔍 Quick Reference: File Content Template

Every file in this repository follows the same 25-section structure:

| # | Section | Purpose |
|---|---------|---------|
| 1 | What is it? | ELI5 + technical definition |
| 2 | Why do we need it? | Problem & motivation |
| 3 | Real-world Example | FAANG production examples |
| 4 | Architecture Diagram | ASCII architecture |
| 5 | Internal Working | Step-by-step explanation |
| 6 | Production Flow | Request lifecycle |
| 7 | HLD | High-level design |
| 8 | LLD | Low-level design |
| 9 | Python Implementation | Production code |
| 10 | Folder Structure | Enterprise layout |
| 11 | Configuration | YAML/JSON config |
| 12 | Flowchart | ASCII flow diagram |
| 13 | Sequence Diagram | ASCII sequence diagram |
| 14 | Pros | Advantages |
| 15 | Cons | Trade-offs |
| 16 | Alternatives | Competitor comparison |
| 17 | Performance | Latency, throughput, GPU |
| 18 | Scaling to Millions | Sharding, caching, autoscaling |
| 19 | Failure Scenarios | Failure modes & mitigation |
| 20 | Security | Auth, PII, prompt injection |
| 21 | Monitoring | Metrics, observability |
| 22 | Interview Questions | All levels |
| 23 | Cheat Sheet | One-page summary |
| 24 | Common Mistakes | Pitfalls |
| 25 | Production Best Practices | Recommendations |

---

## 📊 Comparison Tables

### Framework Comparison
| Feature | LangChain | LangGraph | CrewAI | AutoGen | LlamaIndex |
|---------|-----------|-----------|--------|---------|------------|
| Agent Support | ✅ | ✅ | ✅ | ✅ | Limited |
| State Management | ❌ | ✅ | ✅ | ✅ | ❌ |
| Multi-Agent | ❌ | ✅ | ✅ | ✅ | ❌ |
| Streaming | ✅ | ✅ | ✅ | ✅ | ✅ |
| Production Ready | ⚠️ | ✅ | ⚠️ | ⚠️ | ✅ |
| Learning Curve | Medium | High | Low | Medium | Medium |

### Vector DB Comparison
| Feature | Qdrant | Pinecone | Milvus | ChromaDB | Weaviate |
|---------|--------|----------|--------|----------|----------|
| Self-hosted | ✅ | ❌ | ✅ | ✅ | ✅ |
| Cloud-native | ❌ | ✅ | ✅ | ❌ | ✅ |
| Hybrid Search | ✅ | ✅ | ✅ | ❌ | ✅ |
| Filtering | ✅ | ✅ | ✅ | ✅ | ✅ |
| GPU Acceleration | ❌ | ❌ | ✅ | ❌ | ❌ |
| Distributed | ✅ | ✅ | ✅ | ❌ | ✅ |

### Inference Engine Comparison
| Feature | vLLM | SGLang | Ollama | HF Transformers |
|---------|------|--------|--------|-----------------|
| PagedAttention | ✅ | ✅ | ❌ | ❌ |
| Continuous Batching | ✅ | ✅ | ✅ | ❌ |
| Prefix Caching | ✅ | ✅ | ❌ | ❌ |
| Tensor Parallel | ✅ | ✅ | ❌ | ✅ |
| Quantization | ✅ | ✅ | ✅ | ✅ |
| OpenAI API Compat | ✅ | ✅ | ✅ | ❌ |
| Throughput | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

---

## 🛠️ Tech Stack

| Layer | Technologies |
|-------|-------------|
| **Inference** | vLLM, SGLang, Ollama, HF Transformers |
| **Frameworks** | LangChain, LangGraph, LlamaIndex, CrewAI |
| **Vector DBs** | Qdrant, ChromaDB, Milvus, Pinecone |
| **Backend** | FastAPI, Pydantic, asyncio |
| **Infrastructure** | Docker, Kubernetes, Nginx |
| **Monitoring** | Prometheus, Grafana, OpenTelemetry |
| **MLOps** | MLflow, DVC, Airflow |
| **Storage** | Redis, PostgreSQL, S3 |
| **GPU** | CUDA, TensorRT, vLLM |

---

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

## ⭐ Quick Start

```bash
# Clone the repository
git clone https://github.com/your-org/Agentic-AI-Roadmap.git
cd Agentic-AI-Roadmap

# Pick a module
ls 01_Foundation/

# Start learning
open 01_Foundation/transformers.md
```

---

*Built for engineers who build real AI systems.*
