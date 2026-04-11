---
sprint_number: 1
sprint_name: "C1 System Context"
---

# C1 System Context: LlmSwitch

## Narrative

The LlmSwitch system functions as an intelligent LLM proxy that dynamically selects
optimal language models for AI applications. At the System Context level (C1), we
define the boundary of LlmSwitch and its interactions with external actors and
systems. The system serves two primary human roles: Developers who integrate and
utilize the proxy, and Operations personnel who deploy and maintain it in the
Nomad cluster environment.

LlmSwitch provides OpenAPI and Anthropic Message API-compatible endpoints,
enabling seamless integration with existing AI applications without code changes.
For observability, it exports metrics to Prometheus for monitoring and alerting.
Service discovery and configuration distribution are handled through Consul,
while secure secret management (including API keys) is integrated with Vault.
The system runs as a job in the Nomad orchestration platform for deployment and
lifecycle management.

Note: While LlmSwitch routes requests to LLM services (both local and frontier
models), these are not shown as external containers in this C1 diagram because
they are not listed as external dependencies in the PRD section 'Integrations'.
Instead, LLM services are considered part of the core routing functionality and
will be detailed in the C2 container diagram. The PRD 'Integrations' section
specifically lists: Prometheus, Consul, Vault, and standard AI/ML orchestration
systems (represented as Orchestration). The OpenAPI and Anthropic compatibility
requirements describe LlmSwitch's adherence to API standards, not external
systems, and batch processing is an internal capability.

Each external dependency from PRD 'Integrations' is essential: 
- **Prometheus**: Critical for operational visibility, enabling metrics collection on
  request latency, model usage, and system health to monitor SLA compliance and
  optimize routing decisions.
- **Consul**: Necessary for dynamic service discovery in Nomad, allowing LlmSwitch
  to locate backend services and receive configuration updates without manual
  intervention.
- **Vault**: Mandatory for security compliance, providing secure storage and
  rotation of sensitive credentials like frontier model API keys, preventing
  plaintext secret storage that would violate cluster security policies.
- **Orchestration**: Essential for deployment and management, enabling LlmSwitch
  to run as a managed service in the target infrastructure with automated
  scaling, health checks, and lifecycle management.

## Diagram

```mermaid
C4Context
    title System Context: LlmSwitch

    Person_Ext(Developer, "Developer", "Uses LlmSwitch")
    Person_Ext(Operations, "Operations", "Deploys LlmSwitch")

    System(LlmSwitch, "LlmSwitch", "Intelligent LLM Proxy")

    Container_Ext(Prometheus, "Prometheus", "Monitoring", "Metrics and alerting")
    Container_Ext(Consul, "Consul", "Discovery", "Service discovery")
    Container_Ext(Vault, "Vault", "Secrets", "Secure API key storage")
    Container_Ext(Orchestration, "Orchestration", "Orchestration", "Job scheduling")

    Developer -> LlmSwitch : "Sends LLM requests"
    Operations -> LlmSwitch : "Deploys & manages system"
    LlmSwitch -> Prometheus : "Exports metrics via HTTP"
    LlmSwitch <-> Consul : "DNS/HTTP service discovery"
    LlmSwitch -> Vault : "Gets secrets via AppRole"
    Orchestration -> LlmSwitch : "Health check requests"

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Details

| Relationship | Direction | Protocol | Authentication/Details | PRD Reference |
|--------------|-----------|----------|------------------------|---------------|
| Developer → LlmSwitch | Developer to system | HTTP/HTTPS | Bearer token (API key) | [PRD 6.2] |
| Operations → LlmSwitch | Operations to system | HTTP/HTTPS | API key for admin endpoints | [PRD 6.2] |
| LlmSwitch → Prometheus | System to monitoring | HTTP | None (public metrics endpoint) | [PRD 10.1] |
| LlmSwitch ↔ Consul | Bidirectional | HTTP/DNS | None (Consul agent communication) | [PRD 10.1] |
| LlmSwitch → Vault | System to secret manager | HTTP/HTTPS | AppRole role ID/secret ID | [PRD 10.1] |
| Orchestration → LlmSwitch | Orchestration to system | HTTP/HTTPS | Nomad token or TLS | [PRD 10.1, 12] |

## Element-to-PRD Mapping Table

| Diagram Element | Purpose | PRD Section |
|-----------------|---------|-------------|
| Developer | Integrates and uses LlmSwitch via OpenAPI/Anthropic-compatible APIs | [PRD 5] |
| Operations | Deploys, monitors, and maintains LlmSwitch in Nomad cluster | [PRD 5] |
| LlmSwitch | Intelligent LLM Proxy System providing dynamic model selection | [PRD 1] |
| Prometheus | Metrics collection and alerting for monitoring | [PRD 10.1] |
| Consul | Service discovery and configuration distribution | [PRD 10.1] |
| Vault | Secure secret management for API keys | [PRD 10.1] |
| Orchestration | Job scheduling and management platform for deployment | [PRD 10.1] |

## Rationale for External Dependencies

Each external dependency is critical to LlmSwitch's core value proposition:

1. **Prometheus**: Essential for operational excellence in the cluster environment.
   Without metrics collection, Operations cannot monitor SLA compliance (sub-500ms
   routing decisions), track cost efficiency metrics (local vs frontier usage), or
   set up alerts for anomalous behavior, violating the Non-Functional Requirement
   for observability and the Measurable Outcome for system uptime.

2. **Consul**: Required for dynamic service discovery in Nomad, allowing LlmSwitch
   to locate backend models (both local and frontier) and distribute configuration
   changes across instances. Manual configuration would introduce operational
   overhead, contradicting the Goal of minimal administration and the User Journey
   for simple deployment and maintenance.

3. **Vault**: Mandatory for security compliance. Storing frontier model API keys in
   plaintext would violate cluster security policies. Vault integration enables
   secure credential rotation and fine-grained access control, satisfying the
   Security requirement for secret management and the Integration with Vault for
   secure secret management.

4. **Orchestration**: Necessary for deployment flexibility and operational
   reliability. LlmSwitch must run as a managed service in the target
   infrastructure (Nomad/Kubernetes) to enable automated scaling, health checks,
   and lifecycle management. This supports the Requirement for simple Nomad
   deployment, the Designed to run in Nomad cluster environment specification,
   and the Compatibility with standard AI/ML orchestration and pipeline systems
   integration point.

These dependencies collectively enable LlmSwitch to deliver on its promise:
an intelligent, observable, secure, and operationally simple LLM proxy that
reduces manual model selection overhead while improving cost efficiency
through intelligent routing.