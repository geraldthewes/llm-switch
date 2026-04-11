---
sprint_number: 1
sprint_name: "C1 System Context"
---

# C1 System Context: llm-switch

## Narrative

The llm-switch system functions as an intelligent LLM proxy that dynamically selects optimal language models for AI applications. At the System Context level (C1), we define the boundary of llm-switch and its interactions with external actors and systems. The system serves two primary human roles: Developers who integrate and utilize the proxy, and Operations personnel who deploy and maintain it in the Nomad cluster environment.

llm-switch provides OpenAPI and Anthropic Message API-compatible endpoints, enabling seamless integration with existing AI applications without code changes. For observability, it exports metrics to Prometheus for monitoring and alerting. Service discovery and configuration distribution are handled through Consul, while secure secret management (including API keys) is integrated with Vault. The system also connects to standard AI/ML orchestration platforms for deployment and lifecycle management, and supports batch processing modes common in AI workloads.

Each external dependency is essential: 
- **OpenAPI/Anthropic Specifications**: Necessary for protocol compatibility, allowing llm-switch to act as a drop-in replacement for direct model APIs.
- **Prometheus**: Critical for operational visibility, enabling metrics collection on request latency, model usage, and system health.
- **Consul**: Required for dynamic service discovery in the Nomad cluster, allowing llm-switch to locate backend models and configuration services.
- **Vault**: Mandatory for secure storage and rotation of sensitive credentials like frontier model API keys.
- **AI/ML Orchestration Systems**: Needed for deployment integration, allowing llm-switch to run as a job in platforms like Nomad or Kubernetes.
- **Batch Processing Support**: Required to handle high-throughput AI workloads efficiently without requiring application-level changes.

## Diagram

```mermaid
C4Context
    title System Context: llm-switch

    Person_Ext(Developer, "Developer", "Integrates and uses llm-switch via OpenAPI/Anthropic-compatible APIs [PRD 5]")
    Person_Ext(Operations, "Operations", "Deploys, monitors, and maintains llm-switch in Nomad cluster [PRD 5]")

    System(llm-switch, "llm-switch", "Intelligent LLM Proxy System providing dynamic model selection [PRD 1]")

    Container_Ext(OpenAPI_Spec, "OpenAPI Specification", "Specification", "OpenAPI API compatibility for seamless integration [PRD 10.1]")
    Container_Ext(Anthropic_Spec, "Anthropic Message API", "Specification", "Anthropic Message API compatibility [PRD 10.1]")
    Container_Ext(Prometheus, "Prometheus", "Monitoring System", "Metrics collection and alerting [PRD 10.1]")
    Container_Ext(Consul, "Consul", "Service Discovery", "Service discovery and configuration distribution [PRD 10.1]")
    Container_Ext(Vault, "Vault", "Secret Management", "Secure secret management for API keys [PRD 10.1]")
    Container_Ext(AIML_Orch, "AI/ML Orchestration", "Orchestration System", "Standard AI/ML orchestration and pipeline systems [PRD 10.1]")
    Container_Ext(Batch_Support, "Batch Processing", "Feature", "Support for batch processing modes in AI workloads [PRD 10.1]")

    Developer -> llm-switch : "Sends LLM requests via OpenAPI/Anthropic APIs [PRD 6.2]"
    Operations -> llm-switch : "Deploys, monitors, and maintains system [PRD 6.2]"

    llm-switch -> OpenAPI_Spec : "Implements OpenAPI specification [PRD 10.1]"
    llm-switch -> Anthropic_Spec : "Implements Anthropic Message API specification [PRD 10.1]"
    llm-switch -> Prometheus : "Exports metrics via HTTP [PRD 10.1]"
    llm-switch <-> Consul : "Service discovery via DNS/HTTP, configuration via KV store [PRD 10.1]"
    llm-switch -> Vault : "Retrieves secrets using AppRole authentication [PRD 10.1]"
    llm-switch <-> AIML_Orch : "Integrates via job specifications and API [PRD 10.1]"
    llm-switch -> Batch_Support : "Complies with batch processing standards [PRD 10.1]"

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Details

| Relationship | Direction | Protocol | Authentication/Details | PRD Reference |
|--------------|-----------|----------|------------------------|---------------|
| Developer → llm-switch | Developer to system | HTTP/HTTPS | Bearer token (API key) in Authorization header | [PRD 6.2] |
| Operations → llm-switch | Operations to system | HTTP/HTTPS | API key for administrative endpoints | [PRD 6.2] |
| llm-switch → OpenAPI_Spec | System to specification | N/A | N/A (implementation) | [PRD 10.1] |
| llm-switch → Anthropic_Spec | System to specification | N/A | N/A (implementation) | [PRD 10.1] |
| llm-switch → Prometheus | System to monitoring | HTTP | None (metrics endpoint is public) | [PRD 10.1] |
| llm-switch ↔ Consul | Bidirectional | HTTP/DNS | None (Consul agent communication) | [PRD 10.1] |
| llm-switch → Vault | System to secret manager | HTTP | AppRole role ID/secret ID | [PRD 10.1] |
| llm-switch ↔ AIML_Orch | Bidirectional | HTTP/HTTPS | API key or service token | [PRD 10.1] |
| llm-switch → Batch_Support | System to feature | N/A | N/A (internal capability) | [PRD 10.1] |

## Rationale for External Dependencies

Each external dependency is critical to llm-switch's core value proposition:

1. **OpenAPI/Anthropic Specifications**: These are foundational to llm-switch's ability to provide zero-integration-cost compatibility. By implementing these standards, llm-switch can replace direct model API calls without requiring application modifications, directly enabling the Developer Journey's seamless integration.

2. **Prometheus**: Essential for operational excellence in the cluster environment. Without metrics collection, Operations cannot monitor SLA compliance (sub-500ms routing decisions), track cost efficiency metrics, or set up alerts for anomalous behavior, violating the Non-Functional Requirement for observability.

3. **Consul**: Required for dynamic service discovery in Nomad. llm-switch must locate backend models (both local and frontier) and distribute configuration changes across instances. Manual configuration would introduce operational overhead, contradicting the Goal of minimal administration.

4. **Vault**: Mandatory for security compliance. Storing frontier model API keys in plaintext would violate cluster security policies. Vault integration enables secure credential rotation and fine-grained access control, satisfying the Security requirement for secret management.

5. **AI/ML Orchestration Systems**: Necessary for deployment flexibility. llm-switch must run as a managed service in the target infrastructure (Nomad/Kubernetes). Direct integration with orchestration platforms enables automated scaling, health checks, and lifecycle management, supporting the Requirement for simple Nomad deployment.

6. **Batch Processing Support**: Critical for handling real-world AI workloads. Many AI applications process requests in batches (e.g., embedding generation for document collections). Without native batch support, llm-switch would force applications to implement batching logic, increasing integration complexity and undermining the zero-code-change goal.

These dependencies collectively enable llm-switch to deliver on its promise: an intelligent, observable, secure, and operationally simple LLM proxy that reduces manual model selection overhead while improving cost efficiency through intelligent routing.