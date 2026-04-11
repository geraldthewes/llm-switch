---
sprint_number: 1
sprint_name: "C1 System Context"
---

# C1 System Context: LlmSwitch

## Narrative

The LlmSwitch system functions as an intelligent LLM proxy that dynamically selects optimal language models for AI applications. At the System Context level (C1), we define the boundary of LlmSwitch and its interactions with external actors and systems. This diagram follows the C4 Level 1 specification, showing llm-switch as a single black-box system without exposing internal components.

LlmSwitch serves two primary human roles: Developers who integrate and utilize the proxy through standard APIs, and Operations personnel who deploy and maintain it in the Nomad cluster environment. The system provides OpenAPI and Anthropic Message API-compatible endpoints, enabling seamless integration with existing AI applications without code changes—a key aspect of the Developer Journey section of the PRD.

For operational excellence, LlmSwitch exports metrics to Prometheus for monitoring and alerting, integrates with Consul for service discovery and configuration distribution, and uses Vault for secure secret management of API keys. The system runs as a job in the Nomad orchestration platform, which handles deployment, scaling, health checks, and lifecycle management.

LlmSwitch routes requests to various LLM backend services including local models (Qwen and Nemotron families) and frontier APIs (OpenAI/Anthropic-compatible services). These backend systems are shown as external software systems in this context diagram as required by the sprint contract.

Each external dependency is essential to LlmSwitch's core value proposition:
- **Local Models Qwen/Nemotron**: Provide cost-effective, privacy-preserving inference for suitable workloads
- **Frontier API**: Access to state-of-the-art models for complex tasks requiring maximum capability
- **Nomad Cluster**: Production orchestration environment ensuring reliability and scalability
- **Consul**: Enables dynamic service discovery in distributed environments
- **Vault**: Provides secure credential management meeting security compliance requirements
- **Prometheus**: Delivers observability critical for SLA monitoring and performance optimization

The Operations Journey section of the PRD emphasizes minimal ongoing administration, which is achieved through Nomad's automated management combined with LlmSwitch's self-learning capabilities (detailed in C2/C3 diagrams). The API Backend Specific Requirements section is satisfied through LlmSwitch's protocol-native design that exposes standard OpenAPI/Anthropic endpoints while handling intelligent routing internally.

This system context establishes the foundation for understanding how LlmSwitch integrates into the broader AI infrastructure ecosystem while maintaining a clean separation of concerns between the proxy system and its external dependencies.

## Diagram

```mermaid
C4Context
    title System Context: LlmSwitch

    Person_Ext(Developer, "Developer", "Uses LlmSwitch")
    Person_Ext(Operations, "Operations", "Deploys LlmSwitch")

    System_Boundary(b0, "LlmSwitch") {
        System(LlmSwitch, "LlmSwitch", "Intelligent LLM Proxy")
    }

    System_Ext(LocalModelsQwen, "Local Models Qwen", "LLM Inference", "Qwen language models")
    System_Ext(LocalModelsNemotron, "Local Models Nemotron", "LLM Inference", "Nemotron language models")
    System_Ext(FrontierAPI, "Frontier API", "LLM API", "OpenAI/Anthropic-compatible frontier models")
    System_Ext(NomadCluster, "Nomad Cluster", "Orchestration", "Job scheduling and management")
    System_Ext(Consul, "Consul", "Discovery", "Service discovery and configuration")
    System_Ext(Vault, "Vault", "Secrets", "Secure API key storage")
    System_Ext(Prometheus, "Prometheus", "Monitoring", "Metrics and alerting")

    Developer -> LlmSwitch : "HTTPS API"
    Operations -> LlmSwitch : "Deployment"
    LlmSwitch -> LocalModelsQwen : "gRPC/HTTP"
    LlmSwitch -> LocalModelsNemotron : "gRPC/HTTP"
    LlmSwitch -> FrontierAPI : "HTTPS"
    LlmSwitch -> NomadCluster : "Deployment"
    LlmSwitch -> Consul : "Service Discovery"
    LlmSwitch -> Vault : "Secrets Management"
    LlmSwitch -> Prometheus : "Metrics Export"
    NomadCluster -> LlmSwitch : "Health Check"

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Details

| Relationship | Direction | Protocol | Authentication/Details | PRD Reference |
|--------------|-----------|----------|------------------------|---------------|
| Developer → LlmSwitch | Developer to system | HTTPS | Bearer token (API key) | [PRD 6.2] |
| Operations → LlmSwitch | Operations to system | HTTPS | API key for admin endpoints | [PRD 6.2] |
| LlmSwitch → Local Models Qwen | System to Qwen | gRPC/HTTP | None (internal cluster comms) | [PRD 10.1] |
| LlmSwitch → Local Models Nemotron | System to Nemotron | gRPC/HTTP | None (internal cluster comms) | [PRD 10.1] |
| LlmSwitch → Frontier API | System to Frontier | HTTPS | API key (managed via Vault) | [PRD 10.1] |
| LlmSwitch → Nomad Cluster | System to Nomad | HTTPS | Nomad token or TLS | [PRD 10.1, 12] |
| LlmSwitch → Consul | System to Consul | HTTP/DNS | None (Consul agent communication) | [PRD 10.1] |
| LlmSwitch → Vault | System to Vault | HTTPS/HTTPS | AppRole role ID/secret ID | [PRD 10.1] |
| LlmSwitch → Prometheus | System to Prometheus | HTTP | None (public metrics endpoint) | [PRD 10.1] |
| Nomad Cluster → LlmSwitch | Nomad to system | HTTPS | Nomad token or TLS | [PRD 10.1, 12] |

## Element-to-PRD Mapping Table

| Diagram Element | Purpose | PRD Section |
|-----------------|---------|-------------|
| Developer | Integrates and uses LlmSwitch via OpenAPI/Anthropic-compatible APIs | [PRD 5] Developer Journey |
| Operations | Deploys, monitors, and maintains LlmSwitch in Nomad cluster | [PRD 5] Operations Journey |
| LlmSwitch | Intelligent LLM Proxy System providing dynamic model selection | [PRD 1] Executive Summary |
| Local Models Qwen | Cost-effective local LLM inference for suitable workloads | [PRD 10.1] Integrations |
| Local Models Nemotron | Cost-effective local LLM inference for suitable workloads | [PRD 10.1] Integrations |
| Frontier API | Access to state-of-the-art models for complex tasks | [PRD 10.1] Integrations |
| Nomad Cluster | Production orchestration environment for deployment and scaling | [PRD 10.1] Integrations, [PRD 12] API Backend Specific Requirements |
| Consul | Service discovery and configuration distribution in Nomad | [PRD 10.1] Integrations |
| Vault | Secure secret management for API keys and sensitive credentials | [PRD 10.1] Integrations |
| Prometheus | Metrics collection and alerting for system observability | [PRD 10.1] Integrations |

## Rationale for External Dependencies

Each external dependency is critical to LlmSwitch's core value proposition as defined in the PRD:

1. **Local Models Qwen/Nemotron**: Essential for achieving the cost efficiency goal where "Only send requests to frontier models when no local model can solve the task" (User Success criteria). These models enable privacy-preserving, cost-effective inference while reducing dependence on expensive frontier APIs.

2. **Frontier API**: Provides access to state-of-the-art capabilities for tasks requiring maximum performance, supporting the goal of "Improved system reliability leading to fewer failed AI-assisted tasks" (Business Success).

3. **Nomad Cluster**: Satisfies the Technical Success requirement for being "Designed to run in Nomad cluster environment" and enables "Simple setup and deployment (well packaged)" through job specifications.

4. **Consul**: Required for dynamic service discovery in distributed environments, supporting the Operational Excellence goal of "Clear logging and reporting for troubleshooting and system optimization" and enabling easy integration of new LLM models.

5. **Vault**: Critical for Security requirements including "Encryption of sensitive data at rest (API keys, configuration)" and "Secure integration with Vault for secret management." Prevents plaintext secret storage that would violate cluster security policies.

6. **Prometheus**: Essential for Observability requirements including "Prometheus-compatible metrics endpoint for monitoring key performance indicators" and enabling the Measurable Outcome of "System uptime of 99.9% or better" through proactive monitoring.

7. **Orchestration (Nomad Cluster)**: Directly supports the API Backend Specific Requirements for "Integration with standard AI/ML orchestration and pipeline systems" and provides the "Health check endpoint for cluster orchestration systems" requirement.

These dependencies collectively enable LlmSwitch to deliver on its promise: an intelligent, observable, secure, and operationally simple LLM proxy that reduces manual model selection overhead while improving cost efficiency through intelligent routing—the core innovation described in the Innovation & Novel Patterns section of the PRD.