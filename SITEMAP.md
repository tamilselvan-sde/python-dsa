# Sitemap — Python DSA & AI/ML Roadmap

> Author: [Tamilselvan](https://www.linkedin.com/in/tamilselvan-ai/)
>
> This sitemap lists every document in this repository for search engine and AI crawler discoverability.

---

## Root Level

| File | Description |
|------|-------------|
| [README.md](README.md) | Repository overview, structure, quick start |
| [python-list.md](python-list.md) | Python List DSA — 17 chapters, all list operations, slicing, two pointers, sliding window, monotonic stack, 30 interview questions |
| [python-loops.md](python-loops.md) | Python Loops & Conditionals — if/elif/else, for, while, for-else, iterators, generators, 20 interview questions |
| [SITEMAP.md](SITEMAP.md) | This file — full document index for crawlers |

---

## Agentic AI Roadmap

### Foundation (`Agentic-AI-Roadmap/01_Foundation/`)

| File | Description |
|------|-------------|
| [README.md](Agentic-AI-Roadmap/README.md) | Full curriculum overview, 16-week roadmap |
| [ai_vs_ml_vs_dl.md](Agentic-AI-Roadmap/01_Foundation/ai_vs_ml_vs_dl.md) | AI vs ML vs Deep Learning — hierarchy and distinctions |
| [attention.md](Agentic-AI-Roadmap/01_Foundation/attention.md) | Attention Mechanism — how models focus on relevant input |
| [flash_attention.md](Agentic-AI-Roadmap/01_Foundation/flash_attention.md) | Flash Attention — IO-aware attention algorithm |
| [kv_cache.md](Agentic-AI-Roadmap/01_Foundation/kv_cache.md) | KV Cache — caching Key/Value tensors during decoding |
| [moe.md](Agentic-AI-Roadmap/01_Foundation/moe.md) | Mixture of Experts — sparse activation architecture |
| [positional_encoding.md](Agentic-AI-Roadmap/01_Foundation/positional_encoding.md) | Positional Encoding — sinusoidal position encoding |
| [rope.md](Agentic-AI-Roadmap/01_Foundation/rope.md) | Rotary Position Embedding (RoPE) |
| [self_attention.md](Agentic-AI-Roadmap/01_Foundation/self_attention.md) | Self-Attention — each token attending to all others |
| [speculative_decoding.md](Agentic-AI-Roadmap/01_Foundation/speculative_decoding.md) | Speculative Decoding — draft-verify paradigm |
| [transformers.md](Agentic-AI-Roadmap/01_Foundation/transformers.md) | Transformer Architecture — Attention Is All You Need |

### LLM Inference (`Agentic-AI-Roadmap/02_LLM_Inference/`)

| File | Description |
|------|-------------|
| [continuous_batching.md](Agentic-AI-Roadmap/02_LLM_Inference/continuous_batching.md) | Continuous Batching — dynamic sequence scheduling |
| [hf_transformers.md](Agentic-AI-Roadmap/02_LLM_Inference/hf_transformers.md) | HuggingFace Transformers — unified model API |
| [kv_cache_offloading.md](Agentic-AI-Roadmap/02_LLM_Inference/kv_cache_offloading.md) | KV Cache Offloading — CPU/NVMe cache offload |
| [ollama.md](Agentic-AI-Roadmap/02_LLM_Inference/ollama.md) | Ollama — local LLM runner |
| [paged_attention.md](Agentic-AI-Roadmap/02_LLM_Inference/paged_attention.md) | PagedAttention — virtual memory for KV cache |
| [pipeline_parallel.md](Agentic-AI-Roadmap/02_LLM_Inference/pipeline_parallel.md) | Pipeline Parallelism — layer splitting across GPUs |
| [prefix_cache.md](Agentic-AI-Roadmap/02_LLM_Inference/prefix_cache.md) | Prefix Caching — reusing common prefix KV cache |
| [sglang.md](Agentic-AI-Roadmap/02_LLM_Inference/sglang.md) | SGLang — structured generation framework |
| [tensor_parallel.md](Agentic-AI-Roadmap/02_LLM_Inference/tensor_parallel.md) | Tensor Parallelism — weight matrix splitting |
| [vllm.md](Agentic-AI-Roadmap/02_LLM_Inference/vllm.md) | vLLM — high-performance inference engine |

### RAG (`Agentic-AI-Roadmap/03_RAG/`)

| File | Description |
|------|-------------|
| [rag.md](Agentic-AI-Roadmap/03_RAG/rag.md) | RAG — Retrieval-Augmented Generation overview |
| [bm25.md](Agentic-AI-Roadmap/03_RAG/bm25.md) | BM25 — state-of-the-art sparse retrieval |
| [chunking.md](Agentic-AI-Roadmap/03_RAG/chunking.md) | Document Chunking — splitting strategies |
| [dense_retrieval.md](Agentic-AI-Roadmap/03_RAG/dense_retrieval.md) | Dense Retrieval — neural embedding search |
| [hybrid_search.md](Agentic-AI-Roadmap/03_RAG/hybrid_search.md) | Hybrid Search — combining sparse + dense |
| [metadata_filtering.md](Agentic-AI-Roadmap/03_RAG/metadata_filtering.md) | Metadata Filtering — structured attribute filters |
| [query_rewriting.md](Agentic-AI-Roadmap/03_RAG/query_rewriting.md) | Query Rewriting — preprocessing for better retrieval |
| [reranking.md](Agentic-AI-Roadmap/03_RAG/reranking.md) | Reranking — two-stage retrieval pipeline |
| [semantic_cache.md](Agentic-AI-Roadmap/03_RAG/semantic_cache.md) | Semantic Cache — embedding-based response caching |
| [sparse_retrieval.md](Agentic-AI-Roadmap/03_RAG/sparse_retrieval.md) | Sparse Retrieval — TF-IDF / keyword retrieval |

### Agentic AI (`Agentic-AI-Roadmap/04_Agentic_AI/`)

| File | Description |
|------|-------------|
| [agent.md](Agentic-AI-Roadmap/04_Agentic_AI/agent.md) | AI Agents — reasoning, tool use, memory |
| [autonomous_agents.md](Agentic-AI-Roadmap/04_Agentic_AI/autonomous_agents.md) | Autonomous Agents — long-running AI agents |
| [memory.md](Agentic-AI-Roadmap/04_Agentic_AI/memory.md) | Agent Memory — short-term, long-term, episodic |
| [multi_agent.md](Agentic-AI-Roadmap/04_Agentic_AI/multi_agent.md) | Multi-Agent Systems — collaborative agents |
| [planner.md](Agentic-AI-Roadmap/04_Agentic_AI/planner.md) | Planning — ReAct, Tree-of-Thought, Plan-and-Execute |
| [reflection.md](Agentic-AI-Roadmap/04_Agentic_AI/reflection.md) | Self-Reflection — agents evaluating own outputs |
| [tool_calling.md](Agentic-AI-Roadmap/04_Agentic_AI/tool_calling.md) | Tool Calling / Function Calling |
| [workflow_agents.md](Agentic-AI-Roadmap/04_Agentic_AI/workflow_agents.md) | Workflow Agents — chains, routers, orchestrator |

### Frameworks (`Agentic-AI-Roadmap/05_Frameworks/`)

agno.md, autogen.md, crewai.md, haystack.md, langchain.md, langgraph.md, llamaindex.md, openai_agents_sdk.md, pydantic_ai.md, semantic_kernel.md

### Vector Databases (`Agentic-AI-Roadmap/06_Vector_Databases/`)

chromadb.md, faiss.md, milvus.md, pinecone.md, qdrant.md, weaviate.md

### Architecture (`Agentic-AI-Roadmap/07_Architecture/`)

api_gateway.md, event_driven.md, hld.md, lld.md, microservices.md, scalable_agents.md, scalable_rag.md

### Deployment (`Agentic-AI-Roadmap/08_Deployment/`)

autoscaling.md, docker.md, gpu_scheduling.md, kubernetes.md, monitoring.md, nginx.md, observability.md

### MLOps (`Agentic-AI-Roadmap/09_MLOps/`)

airflow.md, cicd.md, dvc.md, mlflow.md, model_registry.md

### System Design (`Agentic-AI-Roadmap/10_System_Design/`)

ai_assistant.md, chatbot_design.md, document_search.md, enterprise_rag.md, multi_agent_system.md

### FAANG Interview (`Agentic-AI-Roadmap/11_FAANG_Interview/`)

architecture_questions.md, interview_questions.md, troubleshooting.md

---

## Python DSA Cheat Sheet (`Python-DSA-CheatSheet/`)

### Python Basics

variables.md, input_output.md, operators.md, type_casting.md

### Data Types

boolean.md, dictionary.md, list.md, numbers.md, set.md, strings.md, tuple.md

### Control Flow

break_continue.md, for_loop.md, if_else.md, while_loop.md

### Functions

enumerate.md, filter.md, functions.md, lambda.md, map.md, reduce.md, zip.md

### OOP

classes.md, encapsulation.md, inheritance.md, objects.md, polymorphism.md

### Collections

counter.md, defaultdict.md, deque.md, heapq.md

### Algorithms

backtracking.md, bfs.md, binary_search.md, dfs.md, dynamic_programming.md, graph.md, greedy.md, hash_map.md, heap.md, linked_list.md, prefix_sum.md, queue.md, recursion.md, searching.md, sliding_window.md, sorting.md, stack.md, trees.md, trie.md, two_pointers.md, union_find.md

### Time Complexity

big_o.md

### FAANG Patterns

interview_patterns.md

---

## Service Company Preparation (`Python_Service_Company_Coding_Preparation/`)

### Core Guides

Complexity_Cheatsheet.md, DSA_Patterns.md, Interview_Strategy.md, Learning_Roadmap.md, Python_CheatSheet.md

### Company Questions

TCS.md, Infosys.md, Wipro.md, Accenture.md, HCL.md, CTS.md, Deloitte.md

### DSA Patterns

Arrays.md, Backtracking.md, BinarySearch.md, DynamicProgramming.md, Graph.md, HashMap.md, Heap.md, LinkedList.md, Math.md, Queue.md, Recursion.md, SlidingWindow.md, Stack.md, Strings.md, Trees.md, TwoPointers.md

---

## Vector Database Guide (`VectorDB/`)

### Core Parts (1-25)

Part_01_introduction.md through Part_25_cheat_sheets.md

### Extra Topics

Extra_Advanced_Deployment_Patterns.md, Extra_Advanced_Index_Building_FAISS_GPU.md, Extra_Advanced_RAG_Evaluation.md, Extra_Advanced_RAG_Techniques.md, Extra_Advanced_Vector_Search_Patterns.md, Extra_Expert_Interview_Questions_101_120.md, Extra_Monitoring_Observability.md, Extra_Production_Deployment_Checklist.md, Extra_Production_RAG_Evaluation.md, Extra_Production_Runbook.md, Extra_Qdrant_Advanced_Filter_Strategies.md, Extra_Real_World_Vector_Search_Patterns.md, Extra_Security_Best_Practices.md, Extra_Vector_Search_AB_Testing.md, Extra_Vector_Search_Anti_Patterns.md

---

*Total: 187+ documents · Last updated: July 2026*
