# C1 System Context: llm-switch

llm-switch implements a two-part autonomous learning architecture combining real-time intelligent model selection with offline self-learning to continuously improve routing decisions without ongoing manual intervention. This directly supports the user journeys described in PRD Section 4.2: developers integrate by redirecting API calls to llm-switch, benefiting from automatic model selection based on complexity, latency, and cost without requiring code changes; operations engineers deploy in the Nomad cluster using Consul for service discovery and Vault for secret management of API credentials. The system provides a Prometheus metrics endpoint at /metrics for monitoring and integrates with Grafana dashboard llm-switch-overview (dashboard ID: llm-switch-overview). Hardware-aware routing strategies, as specified in supplementary context Section 3.1, use the memory-priority approach to dynamically assign workloads to appropriate GPU resources based on real-time VRAM availability and queue depth, while Kubernetes Node Affinity constraints ensure optimal scheduling of GPU-intensive tasks to prevent resource starvation and maintain consistent performance under varying load conditions. llm-switch supports OpenAI and Anthropic-compatible APIs, enabling seamless integration with existing AI applications through standard endpoints such as /v1/chat/completions and /v1/messages, thus eliminating the need for application-level modifications when switching between different LLM backends. Security boundaries define a trusted zone requiring mTLS for Consul and Vault communications, with frontier API access protected by TLS 1.3 termination at the public internet boundary, ensuring that sensitive credentials and configuration data remain isolated within the cluster's internal network. By dynamically routing requests to local models such as Qwen 7B GGUF and Nemotron-3-22B when feasible, llm-switch reduces frontier model API costs while maintaining response times under 500ms for 95% of requests, achieving a target local model utilization rate of 90% or higher through intelligent load balancing and failover mechanisms. The offline self-learning system analyzes routing decisions overnight to refine orchestration thresholds, progressively increasing local model utilization and improving cost efficiency without manual intervention, utilizing the AutoResearch loop to perform 5-minute training experiments that validate routing improvements before deployment.

```mermaid
C4Context
    title System Context: llm-switch
    Person_Ext(dev, "Developer", "Software developer integrating AI applications")
    Person_Ext(ops, "Operations", "Operations engineer deploying and maintaining systems")
    System_Ext(nomad, "Nomad orchestrator", "Nomad 1.7.0+ cluster orchestrator")
    Boundary(b0, "Trusted Zone (mTLS Required)", "dashed") {
        System_Ext(consul, "Consul service discovery", "Consul 1.16.0+ service mesh")
        System_Ext(vault, "Vault secret management", "Vault 1.15.0+ secret store")
    }
    System_Ext(qwen, "Qwen 7B GGUF", "Local LLM model")
    System_Ext(nemotron, "Nemotron-3-22B", "Local LLM model")
    Boundary(b1, "Public Internet (TLS 1.3 Termination)", "solid") {
        System_Ext(frontierAPI, "OpenAI gpt-4-turbo", "Frontier API accessed via TLS 1.3")
    }
    System(orchestrator, "Orchestrator Model (1B)", "1B parameter model for intent classification")
    System(statRouting, "Statistical Routing", "NormStat/VecStat routing mechanism")
    System(llmSwitch, "llm-switch", "Golang 1.21+ intelligent LLM proxy running in Docker 24.0+ container")
    Rel(dev, llmSwitch, "HTTPS: Developers send OpenAPI/Anthropic requests to llm-switch", "<10ms")
    Rel(ops, llmSwitch, "HTTPS: Operations configure and monitor llm-switch via API", "<10ms")
    Rel(llmSwitch, nomad, "gRPC: llm-switch queries Nomad for model resource allocation", "<20ms")
    Rel(llmSwitch, consul, "DNS/RPC: llm-switch discovers services via Consul", "<5ms")
    Rel(llmSwitch, vault, "TLS: llm-switch retrieves API tokens from Vault", "<10ms")
    Rel(llmSwitch, qwen, "gRPC: llm-switch routes requests to Qwen 7B GGUF based on orchestration decision", "<50ms")
    Rel(llmSwitch, nemotron, "gRPC: llm-switch routes requests to Nemotron-3-22B based on orchestration decision", "<50ms")
    Rel(llmSwitch, orchestrator, "gRPC: llm-switch sends feature vectors for intent classification", "<5ms")
    Rel(llmSwitch, statRouting, "gRPC: llm-switch activates statistical routing for latent space analysis", "<5ms")
    Rel_Back(frontierAPI, llmSwitch, "HTTPS: llm-switch fails over to OpenAI gpt-4-turbo (<100ms latency)", "HTTPS")
    Rel_Back(vault, llmSwitch, "HTTPS: llm-switch refreshes token from Vault on 401 error (<10ms)", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## PRD Traceability Matrix

| Component | PRD Reference | Linked User Journey Steps |
|-----------|---------------|---------------------------|
| llm-switch | Section 4.2, Page 4 | 4.2.1: Maya integrates by changing application endpoint to llm-switch; 4.2.2: Raj deploys llm-switch via Nomad job specification; 4.2.3: System provides core routing capabilities, self-learning, operational excellence, and developer experience as summarized in the journey requirements. |
| Developer | Section 4.2.1, Page 5 | 4.2.1: Maya (developer) integrates AI applications with llm-switch requiring zero code changes; benefits from automatic model selection and overnight self-learning. |
| Operations | Section 4.2.2, Page 6 | 4.2.2: Raj (operations) deploys and maintains llm-switch with minimal ongoing intervention; checks weekly self-learning reports and adds new models via config update. |
| Nomad orchestrator | Section 4.2.2, Page 6 | 4.2.2: llm-switch sends job specification to Nomad orchestrator for deployment; Raj uses Nomad for initial deployment and scaling. |
| Consul service discovery | Section 4.2.2, Page 6 | 4.2.2: llm-switch queries Consul for service discovery to locate models and services; enables dynamic service registration for scaling. |
| Vault secret management | Section 4.2.2, Page 6 | 4.2.2: llm-switch retrieves secrets from Vault for API key management; token refresh on expiry to maintain secure access. |
| Qwen 7B GGUF | Section 4.2.1, Page 5 | 4.2.1: llm-switch routes simple tasks to Qwen 7B GGUF for local inference; 4.2.2: Utilization monitored and balanced by self-learning system to prevent overuse. |
| Nemotron-3-22B | Section 4.2.1, Page 5 | 4.2.1: llm-switch routes moderate tasks to Nemotron-3-22B; 4.2.2: Underutilization detected and corrected overnight by self-learning system to balance load. |
| OpenAI gpt-4-turbo | Section 4.2.1, Page 5 | 4.2.1: llm-switch routes complex tasks to frontier API; 4.2.2: Initial deployment includes frontier API credentials for fallback and high-complexity tasks. |
| Orchestrator Model (1B) | Section 7.0, Page 6 | 4.2.1: Used for intent and complexity classification to enable real-time routing; 4.2.3: Part of the core routing capabilities that analyze task features. |
| Statistical Routing | Section 7.0, Page 6 | 4.2.1: Activates NormStat/VecStat mechanisms for latent space analysis to refine routing decisions; 4.2.3: Part of the core routing capabilities that provide statistical fallback. |