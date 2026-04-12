# C1 System Context: llm-switch

llm-switch operates as an intelligent LLM proxy within a Nomad cluster, integrating with developer workflows, operations infrastructure, and LLM backends. Developers integrate via API change with zero code modifications, while Operations deploys and monitors via Nomad, Consul, and Vault. The system provides OpenAI/Anthropic-compatible endpoints, featuring real-time routing based on task complexity and hardware telemetry, with offline self-learning improving decisions overnight. All interactions occur within a secure cluster environment: Consul and Vault require mTLS (Trusted Zone), frontier API access terminates TLS 1.3 (Public Internet), and llm-switch exposes Prometheus metrics at /metrics for Grafana dashboard llm-switch-overview, employing memory-priority hardware-aware routing and GPU node affinity.

## Mermaid Diagram
```mermaid
C4Context
    title System Context: llm-switch
    Person_Ext(dev, "Developer", "AI application developer integrating via OpenAPI/Anthropic-compatible APIs")
    Person_Ext(ops, "Operations", "DevOps engineer deploying and monitoring in Nomad cluster")
    System_Ext(nomad, "Nomad orchestrator v1.7.0+", "Cluster workload orchestrator for service scheduling")
    System_Ext(qwen, "Qwen 7B GGUF", "Local open-source LLM for efficient inference (GGUF quantized)")
    System_Ext(nemotron, "Nemotron-3-22B", "Local high-parameter LLM for complex reasoning")
    System(orchestrator, "Orchestrator Model (1B) v1.0", "Fine-tuned intent and complexity classifier (Qwen 2.5 0.5B-Instruct)")
    System(statRouting, "Statistical Routing v1.0", "NormStat/VecStat for hardware-aware routing decisions")
    System(llm_switch, "llm-switch v1.0", "Intelligent LLM proxy system (Go 1.21+, Docker 24.0+)")

    Rel(dev, llm_switch, "HTTPS: Developer sends OpenAI-compatible requests to llm-switch", "<50ms")
    Rel(ops, llm_switch, "HTTPS: Operations retrieve metrics and health checks from llm-switch", "<100ms")
    Rel(llm_switch, frontierAPI, "HTTPS: llm-switch forwards complex requests to frontier API", "<2s")
    Rel(llm_switch, nomad, "HTTPS: llm-switch registers service and fetches node allocation from Nomad", "<50ms")
    Rel(llm_switch, qwen, "HTTP: llm-switch routes requests to Qwen 7B GGUF for efficient inference", "<100ms")
    Rel(llm_switch, nemotron, "HTTP: llm-switch routes requests to Nemotron-3-22B for complex reasoning", "<200ms")
    Rel(llm_switch, orchestrator, "gRPC: llm-switch queries orchestrator for task complexity classification", "<10ms")
    Rel(llm_switch, statRouting, "gRPC: llm-switch queries statRouting for hardware telemetry (VRAM, queue depth)", "<10ms")
    Rel_Back(llm_switch, frontierAPI, "HTTPS: llm-switch performs failover routing to frontier API on local model unavailability or latency >100ms", "<2s", $textStyle="dashed")
    Rel_Back(vault, llm_switch, "HTTPS: Vault sends token expiry notification triggering auto-refresh in llm-switch", "<50ms", $textStyle="dashed")

    Boundary(b_trusted, "Trusted Zone (mTLS Required)", "dashed") {
        System_Ext(consul, "Consul service discovery v1.16.0+", "Service discovery and configuration distribution")
        System_Ext(vault, "Vault secret management v1.15.0+", "Secure secret storage and token management")
    }

    Boundary(b_public, "Public Internet (TLS 1.3 Termination)", "solid") {
        System_Ext(frontierAPI, "Frontier API: OpenAI gpt-4-turbo", "Cloud-based LLM API for complex tasks")
    }

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Description
- Developers send OpenAPI-compatible requests to llm-switch via HTTPS with <50ms latency for integration
- Operations retrieve metrics and health checks via HTTPS with <100ms latency for monitoring
- llm-switch forwards complex requests to frontier API via HTTPS with <2s latency for capable model inference
- llm-switch registers services and fetches node allocation from Nomad via HTTPS with <50ms latency for orchestration
- llm-switch routes requests to Qwen 7B GGUF via HTTP with <100ms latency for efficient local inference
- llm-switch routes requests to Nemotron-3-22B via HTTP with <200ms latency for complex reasoning tasks
- llm-switch queries orchestrator via gRPC for task complexity classification with <10ms latency
- llm-switch queries statRouting via gRPC for hardware telemetry (VRAM, queue depth) with <10ms latency
- llm-switch fails over to frontier API via HTTPS with <2s latency (dashed) when local models unavailable or latency exceeds 100ms
- Vault sends token expiry notifications via HTTPS with <50ms latency (dashed) triggering auto-refresh in llm-switch
- Consul and Vault reside in Trusted Zone requiring mTLS for secure service discovery and secret management
- Frontier API resides in Public Internet with TLS 1.3 termination for secure cloud communication

## PRD Traceability Matrix
| Component | PRD Section | Description |
|-----------|-------------|-------------|
| llm-switch | 3.1 | Core system providing intelligent LLM proxy functionality (technology-choices.md) |
| Developer | 4.2.1 | Maya integrates via API change requiring zero code modifications (User Journey) |
| Operations | 4.2.2, 4.2.3 | Raj deploys via Nomad job specification and monitors via health checks; also handles failure recovery and monitoring (User Journey) |
| Nomad orchestrator | 4.1 | Deployment target for llm-switch in cluster environment (Project Scoping) |
| Consul | 4.6 | Service discovery and configuration distribution (Non-Functional Requirements) |
| Vault | 4.6 | Secure secret management for API keys and tokens (Non-Functional Requirements) |
| Qwen 7B GGUF | 3.1 | Local model for efficient inference (technology-choices.md) |
| Nemotron-3-22B | 3.1 | Local model for complex reasoning (technology-choices.md) |
| Frontier API | 4.1 | OpenAI gpt-4-turbo for complex task handling (API Backend Specific Requirements) |
| Orchestrator Model (1B) | 3.1 | Fine-tuned classifier for intent and complexity (technology-choices.md) |
| Statistical Routing | 3.1 | NormStat/VecStat for hardware-aware routing (technology-choices.md) |

All User Journey steps (PRD Sections 4.2.1-4.2.3) are explicitly linked: Section 4.2.3 (Operations monitoring and failure recovery) is represented by the Operations persona interacting with llm-switch for metrics, health checks, and failure recovery via fallback mechanisms.