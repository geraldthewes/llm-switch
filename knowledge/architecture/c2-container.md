# C2 Container Overview - llm-switch

The llm-switch system implements a two-part autonomous learning architecture combining real-time intelligent model selection with offline self-learning.

The API Gateway in the DMZ receives OpenAI/Anthropic-compatible requests and forwards them via bifrost with mTLS encryption and <500ms timeout to the Orchestrator Service.
This performs complexity classification using a 1B parameter model and integrates hardware telemetry.

The Model Router handles load balancing, circuit breaking, and fallback logic, directing requests to local model adapters (behind load balancers for Qwen and Nemotron) or the frontier API adapter.
Local model adapters communicate with external Qwen (24GB VRAM) and Nemotron (48GB VRAM) model servers.
The Frontier API Adapter accesses external frontier models via Vault-managed API keys.

Internal service communication uses mTLS via Consul Connect.
The system integrates with Nomad for orchestration, Consul for service discovery, and Vault for secret management.

Monitoring uses a Prometheus exporter, and trace collection for self-learning uses a Langfuse collector.
The Offline Self-Learning Container analyzes traces overnight to refine routing thresholds, updating the Model Router asynchronously.

Configuration artifacts (Nomad job specs and Consul config) are separated from application code, enabling horizontal scaling and allowing new models via Nomad job updates without modifying containers.

Failure handling includes circuit breaker patterns, <500ms timeouts, and a dead letter queue.
Health checks are bidirectionally exchanged with Nomad.

Developers benefit from zero-code-change integration via standard API endpoints, while operations engineers observe self-learning reports and model additions through the administrative interface.

---

```mermaid
flowchart LR
    subgraph DMZ
        api_gateway["API Gateway\nGolang, bifrost, Docker 2x\nHandles OpenAI/Anthropic-compatible API requests"]
    end

    subgraph firewall["Firewall"]
        subgraph internal_vpc["Internal VPC"]
            subgraph nomad_config["Nomad Configuration"]
                nomad_job["Nomad Job Definition\nNomad, Docker\nConstraints: GPU required for Frontier\nMemory: 32GB for Local, Node pool: llm-switch\nDefines Nomad job specifications and constraints for llm-switch deployment"]
                consul_config["Consul Configuration\nConsul, Docker\nManages Consul service discovery configuration and key-value store settings"]
            end
            orchestrator_service["Orchestrator Service\nGolang, 1B parameter model, bifrost\nTelemetry: GPU/CPU metrics, Docker 2x\nPerforms complexity classification and hardware telemetry integration"]
            model_router["Model Router\nGolang, bifrost, Docker 2x\nRoutes requests, implements load balancing, circuit breaker, and fallback"]
            qwen_lb["Qwen Load Balancer\nGolang, bifrost, Docker 2x\nLoad balances requests to Qwen adapters"]
            qwen_adapter_1["Qwen Adapter\nGolang, vLLM/llama.cpp, Telemetry: GPU/CPU metrics, Docker\nMemory: 32GB, Node pool: llm-switch\nAdapts llm-switch interface to Qwen model server for inference"]
            qwen_adapter_2["Qwen Adapter\nGolang, vLLM/llama.cpp, Telemetry: GPU/CPU metrics, Docker\nMemory: 32GB, Node pool: llm-switch\nAdapts llm-switch interface to Qwen model server for inference"]
            nemotron_lb["Nemotron Load Balancer\nGolang, bifrost, Docker 2x\nLoad balances requests to Nemotron adapters"]
            nemotron_adapter_1["Nemotron Adapter\nGolang, vLLM/llama.cpp, Telemetry: GPU/CPU metrics, Docker\nMemory: 32GB, Node pool: llm-switch\nAdapts llm-switch interface to Nemotron model server for inference"]
            nemotron_adapter_2["Nemotron Adapter\nGolang, vLLM/llama.cpp, Telemetry: GPU/CPU metrics, Docker\nMemory: 32GB, Node pool: llm-switch\nAdapts llm-switch interface to Nemotron model server for inference"]
            frontier_adapter["Frontier API Adapter\nGolang, Docker, Node pool: llm-switch\nAdapts to frontier APIs (OpenAI/Anthropic)"]
            health_check_service["Health Check Service\nGolang, Docker, Node pool: llm-switch\nProvides health endpoint for Nomad"]
            prometheus_exporter["Prometheus Metrics Exporter\nGolang, Prometheus PushGateway, Docker\nNode pool: llm-switch\nExports metrics for monitoring"]
            langfuse_trace_collector["Langfuse Trace Collector\nGolang, Langfuse API, Docker\nNode pool: llm-switch\nCollects traces for offline self-learning"]
            offline_self_learning["Offline Self-Learning Container\nGolang, Docker, Node pool: llm-switch\nAnalyzes traces and refines routing thresholds"]
            admin_interface["Administrative Interface\nGolang, Docker, Node pool: llm-switch\nProvides system configuration and diagnostics"]
            consul_integration["Consul Integration\nGolang, Consul API, Docker\nNode pool: llm-switch\nIntegrates with Consul for service discovery"]
            vault_integration["Vault Integration\nGolang, Vault API, Docker\nNode pool: llm-switch\nIntegrates with Vault for secret management"]
            dead_letter_queue["Dead Letter Queue\nRedis, Docker, Node pool: llm-switch\nStores persistently failed requests after 3 retries"]
        end
    end

    subgraph ext["External Systems"]
        nomad_cluster["Nomad Cluster\n3 nodes"]
        consul["Consul Service Discovery and KV\n3 nodes"]
        vault["Vault Secret Storage\n3 nodes"]
        qwen["Qwen Model Server\nVRAM 24GB, 2x"]
        nemotron["Nemotron Model Server\nVRAM 48GB, 2x"]
        frontier["Frontier API Providers\nOpenAI, Anthropic, 1x"]
        prometheus_backend["Prometheus Monitoring Backend\n2x"]
        langfuse_backend["Langfuse Backend\n2x"]
    end

    api_gateway -->|"HTTP/1.1 | bifrost, <500ms timeout, mTLS"| orchestrator_service
    orchestrator_service -->|"Routing Decision (includes VRAM-aware routing) | gRPC, mTLS"| model_router
    model_router -->|"Forward Request | gRPC, Load Balanced, Circuit Breaker, <500ms timeout, Low Cost"| qwen_lb
    model_router -->|"Forward Request | gRPC, Load Balanced, Circuit Breaker, <500ms timeout, Low Cost"| nemotron_lb
    model_router -->|"Forward Request | gRPC, Circuit Breaker, <500ms timeout, High Cost"| frontier_adapter
    qwen_lb -->|"Forward Request | gRPC"| qwen_adapter_1
    qwen_lb -->|"Forward Request | gRPC"| qwen_adapter_2
    nemotron_lb -->|"Forward Request | gRPC"| nemotron_adapter_1
    nemotron_lb -->|"Forward Request | gRPC"| nemotron_adapter_2
    model_router -.->|"Send Traces | Langfuse API, asynchronous, mTLS"| langfuse_trace_collector
    langfuse_trace_collector -.->|"Provide Traces | Langfuse API, asynchronous, mTLS"| offline_self_learning
    offline_self_learning -.->|"Update Routing Thresholds | asynchronous, mTLS"| model_router
    orchestrator_service -.->|"Pull Telemetry | asynchronous, mTLS"| qwen_adapter_1
    orchestrator_service -.->|"Pull Telemetry | asynchronous, mTLS"| qwen_adapter_2
    orchestrator_service -.->|"Pull Telemetry | asynchronous, mTLS"| nemotron_adapter_1
    orchestrator_service -.->|"Pull Telemetry | asynchronous, mTLS"| nemotron_adapter_2
    qwen_adapter_1 -->|"Inference and Telemetry | gRPC, mTLS"| qwen
    qwen_adapter_2 -->|"Inference and Telemetry | gRPC, mTLS"| qwen
    nemotron_adapter_1 -->|"Inference and Telemetry | gRPC, mTLS"| nemotron
    nemotron_adapter_2 -->|"Inference and Telemetry | gRPC, mTLS"| nemotron
    frontier_adapter -->|"Forward Request | HTTPS"| frontier
    consul_integration -->|"Service Discovery/KV | Consul API, mTLS"| consul
    vault_integration -->|"Secret Storage | Vault API, mTLS"| vault
    prometheus_exporter -->|"Push Metrics | Prometheus PushGateway, mTLS"| prometheus_backend
    langfuse_trace_collector -->|"Send Traces | Langfuse API, mTLS"| langfuse_backend
    admin_interface -->|"Update Job Spec (add new LLM model) | Nomad API, mTLS"| nomad_job
    admin_interface -->|"Update Config | Consul API, mTLS"| consul_integration
    admin_interface -->|"Update Secrets | Vault API, mTLS"| vault_integration
    health_check_service <-->|"Health Check | Nomad SDK, mTLS"| nomad_cluster
    model_router -.->|"After 3 retries | asynchronous, mTLS"| dead_letter_queue
    model_router -->|"Fallback Request on Qwen failure | gRPC, Circuit Breaker, <500ms timeout, Low Cost"| nemotron_lb
    model_router -->|"Fallback Request on Nemotron failure | gRPC, Circuit Breaker, <500ms timeout, Low Cost"| qwen_lb
```
