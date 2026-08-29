# Modern AI-Enabled Enterprise Cloud Architecture — Full Tech Stack Reference
*FAANG System Design Interview Prep — Production-Ready Stack (2026)*
https://www.youtube.com/watch?v=fhdPyoO6aXI&list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM

> Persona: Principal/Staff Software Architect designing a cloud-native, agentic-AI-enabled enterprise platform. Each layer lists the tools/frameworks a real architect would shortlist, with a "pick + why" so you can defend choices in an interview.

---

## 1. Client / Experience Layer
| Category | Technologies |
|---|---|
| Web frontend | React, Next.js (SSR/ISR), TypeScript, Remix, Vue/Nuxt (alt) |
| Mobile | Flutter, React Native, Swift/Kotlin (native) |
| Design system | Tailwind CSS, shadcn/ui, Storybook |
| Real-time UX | WebSockets, Server-Sent Events (SSE), GraphQL Subscriptions |
| State/data fetching | TanStack Query, Redux Toolkit, Zustand |
| Streaming AI UX | Vercel AI SDK (streaming tokens to UI), react-markdown for chat rendering |

**Interview talking point:** SSE/streaming is now standard for LLM token-by-token UX; WebSockets reserved for bidirectional agent/tool-call scenarios.

---

## 2. Edge, DNS, CDN & Network
| Category | Technologies |
|---|---|
| DNS | Route 53, Cloudflare DNS, Azure DNS |
| CDN / Edge compute | CloudFront, Cloudflare (Workers), Fastly, Azure Front Door |
| Edge AI Gateway | Cloudflare AI Gateway, Vercel AI Gateway (edge-native LLM proxying/caching) |

## 3. Security Perimeter
| Category | Technologies |
|---|---|
| WAF / DDoS | AWS WAF & Shield, Azure WAF, Cloudflare WAF |
| Bot/abuse mgmt | Cloudflare Bot Management, Arkose Labs |
| Load balancing | ALB/NLB, Azure Load Balancer, GCP Cloud LB, L4/L7 routing |

## 4. Reverse Proxy / API Gateway
| Category | Technologies |
|---|---|
| API Gateway | NGINX, Envoy, Kong, Apigee, AWS API Gateway, Azure API Management |
| **AI-specific gateways** | **Kong AI Gateway** (mesh-native, plugin model, SSO), **TrueFoundry AI Gateway** (1000+ models, VPC-hosted, MCP Gateway for agent/tool traffic) |

---

## 5. Application / Backend Services
| Category | Technologies |
|---|---|
| Languages/frameworks | Python (FastAPI, Django), Node.js (NestJS), Java/Spring Boot, Go (Gin/Fiber), .NET |
| API paradigms | REST, GraphQL (Apollo, Hasura), gRPC, tRPC |
| Compute | VMs, Docker, Kubernetes, ECS/Fargate, AKS, GKE, serverless (Lambda, Cloud Functions, Azure Functions) |

---

## 6. AI / Foundation Model Layer
| Category | Technologies |
|---|---|
| Foundation model providers | Anthropic (Claude), OpenAI (GPT), Google (Gemini), Meta (Llama), Mistral, Cohere |
| Open-weight self-hosted | Llama 3/4, Mistral/Mixtral, Qwen, DeepSeek |
| Fine-tuning / adaptation | LoRA / QLoRA, PEFT, Hugging Face Transformers + TRL, Axolotl, RLHF/DPO pipelines |
| Multi-modal | GPT-4o/Gemini multimodal, Whisper (speech), CLIP/SigLIP (vision-text) |

**Sub-area: Model Selection Criteria (interview depth)**
- Latency/throughput needs → dedicated inference vs. managed API
- Data residency/compliance → self-hosted open-weight vs. SaaS API
- Cost per token vs. quality tradeoff → routed via AI Gateway policies

---

## 7. Model Serving / Inference Layer (deep-dive sub-area)
| Category | Technologies | Notes |
|---|---|---|
| LLM-focused serving engines | **vLLM** (PagedAttention, continuous batching, broadest ecosystem — default choice), **SGLang** (prefix-reuse/RadixAttention, best for shared-prefix workloads) | Where vLLM and SGLang are LLM-focused serving engines, Triton is a general-purpose inference platform that supports multiple frameworks and model types |
| Hardware-optimized engines | **TensorRT-LLM** (NVIDIA-only, compiled engines, lowest latency, speculative decoding) | TensorRT-LLM supports speculative decoding, which delivers up to 3.6x faster generation, and connects natively to Triton, NVIDIA NIM, and NVIDIA Dynamo |
| General-purpose inference platform | NVIDIA Triton (renamed **NVIDIA Dynamo Triton**), KServe, Ray Serve | Multi-framework (PyTorch/TF/ONNX/TensorRT), model versioning, ensembles |
| Distributed/datacenter-scale | **NVIDIA Dynamo** — successor to Triton for disaggregated prefill/decode serving and KV-cache-aware routing | Dynamo's headline ideas are disaggregated serving — splitting a model's prefill and decode phases onto different GPUs — and KV-cache-aware routing |
| Local/dev-only | Ollama, llama.cpp, LM Studio | Single-user, no continuous batching — not for production multi-user traffic |
| Legacy/maintenance | Hugging Face TGI | TGI moved to maintenance mode in March 2026 and now directs new users to vLLM, SGLang, llama.cpp, and MLX |

**Decision framework to state in interview:** start on vLLM for flexibility → move to SGLang if prefix-heavy (chat with long shared system prompts) → move to TensorRT-LLM + Triton/Dynamo only if you're NVIDIA-committed and need the last 15–30% throughput at massive scale.

## 8. GPU / Accelerator Infrastructure
| Category | Technologies |
|---|---|
| Compute | NVIDIA CUDA, A100 / H100 / H200 / B200 (Blackwell), AMD MI300X, Google TPU v5/v6 |
| Scheduling/sharing | MIG (Multi-Instance GPU), Kubernetes device plugins, Run:ai, SLURM (batch/HPC) |
| Distributed training | DeepSpeed, Megatron-LM, PyTorch FSDP, Ray Train |

## 9. AI Gateway / Model Routing (deep-dive sub-area)
| Tool | Positioning |
|---|---|
| **LiteLLM** | Open-source, self-hosted, OpenAI-compatible proxy to 100+ providers; virtual-key budgeting; best default for self-hosted + multi-cloud mix. Handles authentication via virtual keys, rate limiting, per-team budget enforcement, request logging, and automatic failover across providers |
| **Portkey** | Managed SaaS; semantic caching, guardrails, prompt versioning out of the box. Connects to 250+ LLMs through a single endpoint and extends the basic gateway model with observability, guardrails, prompt versioning, and semantic caching |
| **Kong AI Gateway** | Extends existing Kong API-management mesh with LLM plugins; best if already Kong-based. Kong's AI plugins include rate limiting, request transformation, and response filtering, composed rather than configured as a monolith |
| **TrueFoundry** | Treats models/agents/tools as first-class infra objects inside your own VPC; includes an MCP Gateway for agent-tool traffic — strongest enterprise data-sovereignty story. |
| **Cloudflare AI Gateway / Vercel AI Gateway** | Edge-native, zero-infra caching + analytics for web-first stacks. |
| **OpenRouter** | Fastest zero-setup path to 400+ hosted models — good for prototyping, not for regulated enterprises. |

**Interview framing:** the AI Gateway is the FinOps + governance control plane — cost tracking, per-team budgets, automatic provider failover, and audit trail for compliance sit here, not in application code.

---

## 10. Agents / Orchestration Layer (deep-dive sub-area)
| Framework | Best for | Notes |
|---|---|---|
| **LangGraph** (LangChain) | Stateful, production, graph-based multi-agent pipelines with durable execution & human-in-the-loop | Has the largest production deployment footprint of any agent framework in 2026 and is the default runtime for LangChain agents, supporting both Python and JavaScript |
| **CrewAI** | Fast, role-based multi-agent prototyping ("Researcher/Reviewer/Developer" personas) | CrewAI organizes role-based crews with process types, but has no built-in checkpointing for long-running workflows and coarse-grained error handling; teams often migrate to LangGraph for production state management |
| **Microsoft Agent Framework** | Microsoft-stack enterprises | Merges the enterprise features of Semantic Kernel (session state, type safety, middleware, telemetry) with the multi-agent orchestration of AutoGen into one SDK, shipping native MCP + A2A support |
| **AutoGen/AG2** | Conversational multi-agent debate/refinement patterns | Multi-turn GroupChat style coordination |
| **OpenAI Agents SDK / Claude Agent SDK / Google ADK** | Provider-native agent SDKs | Each major lab now ships its own agent dev kit |
| **LlamaIndex Workflows** | Retrieval-first agents, superior RAG pipeline (hybrid search, reranking, self-correction) | Often paired: LlamaIndex for retrieval → LangGraph for orchestration |
| **Semantic Kernel** | .NET/enterprise plugin-model agents (now merging into MS Agent Framework) | |
| **Dify** | Low-code/no-code agent workflow builder for non-technical teams | |
| Protocol layer | **MCP (Model Context Protocol)** — agent-to-tool standard; **A2A (Agent2Agent)** — agent-to-agent interoperability protocol | MCP is becoming the REST API of agent-tool communication |

**Production pattern to quote in interview:** the best production systems use 2–3 frameworks, each handling the layer it was designed for — e.g., CrewAI for fast role-based synthesis feeding into LangGraph for deterministic, auditable execution, compliance review, and human approval. Also note: roughly 28% of production multi-agent deployments still use custom orchestration with no framework at all — framework choice matters less than the evaluation pipeline, observability, and failure-recovery logic.

---

## 11. RAG / Knowledge Layer (deep-dive sub-area)
| Sub-component | Technologies |
|---|---|
| Chunking / parsing | LlamaIndex, Unstructured.io, LangChain text splitters |
| Embeddings | OpenAI text-embedding-3, Cohere Embed, Voyage AI, BGE / Nomic (open-source) |
| Reranking | Cohere Rerank, cross-encoders (BGE-reranker), ColBERT late-interaction |
| Hybrid search | BM25 + vector fusion (reciprocal rank fusion), SPLADE sparse vectors |
| GraphRAG / Knowledge Graphs | Microsoft GraphRAG, Neo4j, **ArangoDB** (multi-model graph+document) |
| Retrieval evaluation | RAGAS, TruLens, DeepEval |

---

## 12. Vector / Graph / Search Databases (deep-dive sub-area)
| Database | Best for | Notes |
|---|---|---|
| **Pinecone** | Fully managed, zero-ops, enterprise SLA | Zero-ops managed search at any scale, now with built-in inference (embeddings + reranking), full-text hybrid search, and BYOC deployments. Weakness: no self-hosting, period; serverless recall is fixed at roughly 90% with no way to tune it |
| **Weaviate** | Hybrid search champion (vector + BM25 + filters), auto-embedding modules | Weaviate can generate embeddings itself on ingestion using built-in vectorizer modules, and led on hybrid search with BlockMAX WAND keyword scoring and reciprocal rank fusion |
| **Milvus / Zilliz Cloud** | Billion-scale, distributed, highest write throughput | Milvus handles the highest write throughput due to its distributed architecture; requires engineering resources to operate |
| **Qdrant** | Rust-based performance, best filtered search, best free tier | Best free tier, native sparse (SPLADE, miniCOIL) and ColBERT multi-vector support |
| **pgvector / pgvectorscale** | Already-Postgres shops, <100M vectors, transactional consistency | Gives SQL filtering, joins, and transactional consistency between documents and embeddings |
| **ChromaDB** | Prototyping/MVP → lightweight production | Now has object-storage backend and collection forking for lightweight production use |
| **Vespa** | Billion-scale hybrid search with tensor ops/learned ranking | |
| **ArangoDB / Neo4j** | Multi-model graph + document + knowledge graph (GraphRAG) | Positioned between transactional DB and vector DB — powers entity/relationship reasoning for agents |
| OpenSearch/Elasticsearch | Vector + full-text + log search combined | |

**Decision framework:** the choice of vector database usually matters less than embedding model quality, chunking strategy, retrieval evaluation, and reranking — a cross-encoder rerank step often improves quality more than switching vector DBs.

---

## 13. Operational Databases & Storage
| Category | Technologies |
|---|---|
| Relational | PostgreSQL, MySQL, SQL Server, CockroachDB (distributed SQL) |
| Document / NoSQL | MongoDB, DynamoDB, Cosmos DB |
| Cache / in-memory | Redis, Memcached |
| Graph | Neo4j, ArangoDB, Amazon Neptune |
| Object/Data Lake | S3, Azure Blob, GCS, Delta Lake, Apache Iceberg, Parquet |

## 14. Messaging / Event-Driven & Workflow
| Category | Technologies |
|---|---|
| Streaming/event bus | Kafka, Confluent, Event Hubs, GCP Pub/Sub, RabbitMQ, AWS SQS/SNS |
| Durable workflow orchestration | Temporal, Apache Airflow, AWS Step Functions, Dagster |

---

## 15. MLOps / LLMOps (deep-dive sub-area)
| Category | Technologies |
|---|---|
| Experiment tracking / model registry | MLflow, Weights & Biases (W&B), Kubeflow |
| Prompt/version management | LangSmith Hub, Portkey Prompt Registry, Langfuse Prompt Management |
| Feature stores | Feast, Tecton |
| CI/CD for models | GitHub Actions + model registry hooks, Seldon Core, BentoML |
| Fine-tuning infra | Axolotl, Hugging Face AutoTrain, TRL, Ray Train, TorchTune |

---

## 16. AI Evaluation & Observability (deep-dive sub-area)
| Tool | Category | Notes |
|---|---|---|
| **Langfuse** | Open-source (MIT), tracing + prompt mgmt + eval + datasets | The most widely adopted open source LLM-specific observability platform; acquired by ClickHouse, signalling strong long-term investment in its data infrastructure |
| **LangSmith** | LangChain/LangGraph-native tracing | Provides the tightest integration and best trace visualization within the LangChain ecosystem, though evaluation depth outside LangChain is more limited |
| **Arize Phoenix** | OpenTelemetry-native, vendor-neutral | Brings OpenTelemetry-native observability to LLM evaluation; open source with 10,000+ GitHub stars, accepting traces via standard OTLP so you avoid vendor lock-in |
| **Braintrust, W&B Weave, AgentOps** | Eval-first / experimentation platforms | Strong for prototyping + non-technical stakeholder review |
| **Helicone** | Fastest-setup logging + gateway + cost tracking | Offers the fastest setup with automatic cost tracking |
| **Confident AI / DeepEval, RAGAS, TruLens** | Open eval libraries/metrics | Focus on scoring outputs — faithfulness, hallucination, answer relevance, and task completion — often via LLM-as-a-judge |
| Infra correlation | Datadog LLM Observability, OpenTelemetry, Prometheus, Grafana, New Relic | Bolt LLM tracing onto existing infrastructure monitoring so AI signals correlate with CPU, memory, and network metrics |

**Key eval dimensions to name in interview:** faithfulness (is the output grounded in context), relevance (does it answer the question), and safety (freedom from toxicity, bias, or PII leakage); for RAG add context relevance and answer correctness.

---

## 17. AI Safety / Guardrails (deep-dive sub-area)
| Tool | Function |
|---|---|
| **NVIDIA NeMo Guardrails** | Programmable rail system (Colang DSL) for dialog flow, jailbreak detection, fact-checking, content safety | NeMo Guardrails commonly calls Llama Guard the classifier behind its content-safety rails, combining broad orchestration with a purpose-built safety model |
| **Llama Guard / Llama Firewall** | Open-weight safety classifier (input/output content moderation) | Supports fine-tuning via LoRA adapters for domain-specific risks and can be deployed on-premise for organizations concerned about data leakage |
| **LLM Guard** | Open-source Python scanners (15 input / 20 output scanners) | |
| **OpenAI Guardrails / Guardrails AI (RAIL spec)** | Structured output validation, tripwire mechanisms | |
| **Lakera Guard** | Commercial managed guardrail API, sub-50ms latency | |
| **Promptfoo** | Red-teaming / vulnerability testing in CI/CD | Testing-first framework with 50+ vulnerability types, real-time guardrails, and CI/CD integration |
| **Granite Guardian, Qwen3Guard, WildGuard, ShieldGemma** | Alternative open safety-classifier models | |

**Architecture note:** guardrails split into **content-safety** (Llama Guard, NeMo) vs. **execution-safety** (tool-call authorization, agent action gating) — a complete agent governance stack integrates both content safety (preventing harmful text generation) and execution safety (preventing unauthorized tool invocations).

---

## 18. Security / Governance / FinOps
| Category | Technologies |
|---|---|
| Identity | IAM, OAuth2/OIDC, Okta, Azure AD, SSO |
| Secrets management | HashiCorp Vault, AWS Secrets Manager |
| PII protection | Presidio, DLP tooling |
| Compliance/audit | Immutable audit logs, SOC 2 / HIPAA / GDPR controls |
| Cost/token tracking (AI FinOps) | Gateway-native budgets (LiteLLM/Portkey), CloudNuro, per-team quota enforcement |

## 19. CI/CD & Infra-as-Code
| Category | Technologies |
|---|---|
| CI/CD | GitHub Actions, GitLab CI, Azure DevOps, Jenkins |
| IaC | Terraform, Pulumi, CloudFormation |
| Deployment | Helm, Argo CD, Argo Rollouts, blue/green & canary strategies |

---

## 20. Reference Architecture Diagram (narrative)

```
Client (React/TS/Next.js) 
   → Edge/CDN (Cloudflare/CloudFront) 
   → WAF/LB 
   → API Gateway (Kong/Envoy) 
   → Backend Services (FastAPI/Node/Go, K8s) 
   → AI Gateway (LiteLLM/Portkey) — routing, budgets, fallback
   → Agent Orchestration (LangGraph + CrewAI + MCP)
   → RAG Layer (chunking/embeddings/rerank)
   → Vector/Graph DB (Qdrant/Weaviate/ArangoDB) + Operational DB (Postgres)
   → Model Serving (vLLM/SGLang/TensorRT-LLM on Triton/Dynamo, GPU: H100/B200)
   → Foundation Models (Claude/GPT/Gemini/Llama)
Cross-cutting: Guardrails (NeMo/Llama Guard) · Observability (Langfuse/Phoenix) · MLOps (MLflow/W&B) · Security/IAM · CI/CD
```

---

## 21. Interview Prep Checklist
- [ ] Justify vLLM vs. TensorRT-LLM vs. managed API tradeoffs (cost, latency, control)
- [ ] Explain why an AI Gateway sits between app and model providers (governance, fallback, cost)
- [ ] Defend a vector DB choice against scale/latency/hybrid-search requirements
- [ ] Describe a 2-framework agent pattern (e.g., CrewAI → LangGraph) and why
- [ ] Name the 3 pillars of LLM eval (faithfulness, relevance, safety) and how you'd instrument tracing (OpenTelemetry)
- [ ] Distinguish content-safety guardrails from execution-safety/tool-authorization guardrails
- [ ] Explain MCP vs. A2A protocol roles in a multi-agent system
- [ ] Walk through disaggregated prefill/decode serving (NVIDIA Dynamo) for a whiteboard bonus point




