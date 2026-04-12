# C1 System Context: llm-switch

llm-switch implements a two-part autonomous learning architecture as explicitly described in PRD Section 4.2 User Journeys, combining real-time intelligent model selection with offline self-learning to continuously improve routing decisions without requiring ongoing manual intervention from developers or operations teams. Developers like Maya integrate the system by simply redirecting API calls to llm-switch, benefiting from automatic routing that selects optimal models per query based on complexity, latency, and cost without requiring code changes, allowing her team to focus on feature development rather than model management as detailed in PRD Section 4.2.1. Operations engineers like Raj deploy llm-switch in the Nomad cluster using a simple job specification, relying on Consul for service discovery to dynamically locate backend models and Vault for secure secret management of API credentials, reducing operational overhead by approximately 70% compared to manual model management approaches as described in PRD Sections 4.2.2-4.2.3. The system provides a Prometheus metrics endpoint at /metrics for monitoring key performance indicators such as request latency, model utilization rates, and error rates, and integrates with Grafana dashboard llm-switch-overview (dashboard ID: llm-switch-overview) for real-time operational visibility, alerting, and historical trend analysis. Hardware-aware routing strategies, as specified in supplementary context Section 3.1, prioritize memory utilization to dynamically assign workloads to appropriate GPU resources based on real-time VRAM availability and queue depth, while Kubernetes Node Affinity constraints ensure optimal scheduling of GPU-intensive tasks to prevent resource starvation, maintain consistent performance under varying load conditions, and enable efficient scaling of AI workloads across the cluster. llm-switch supports OpenAI and Anthropic-compatible APIs, enabling seamless integration with existing AI applications through standard endpoints such as /v1/chat/completions and /v1/messages, thus eliminating the need for application-level modifications when switching between different LLM backends. Security boundaries define a trusted zone requiring mTLS for Consul and Vault communications, with frontier API access protected by TLS 1.3 termination at the public internet boundary, ensuring that sensitive credentials and configuration data remain isolated within the cluster's internal network. By dynamically routing requests to local models such as Qwen 7B GGUF and Nemotron-3-22B when feasible, llm-switch reduces frontier model API costs while maintaining response times under 500ms for 95% of requests, achieving a target local model utilization rate of 90% or higher through intelligent load balancing and failover mechanisms. The offline self-learning system analyzes routing decisions overnight to refine orchestration thresholds, progressively increasing local model utilization and improving cost efficiency without manual intervention, utilizing the AutoResearch loop to perform 5-minute training experiments that validate routing improvements before deployment.

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
    System_Ext(frontierAPI, "OpenAI gpt-4-turbo", "Frontier API accessed via TLS 1.3")
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
    Rel_Back(llmSwitch, frontierAPI, "Failover on latency >100ms", "<2s")
    Rel_Back(vault, llmSwitch, "401 → Token Refresh", "<1s")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## PRD Traceability Matrix

| Component | PRD Reference |
|-----------|---------------|
| llm-switch | Section 4.0, Page 3 |
| Developer | Section 4.2.1, Page 5 |
| Operations | Section 4.2.2, Page 5 |
| Nomad orchestrator | Section 9.0, Page 8 |
| Consul service discovery | Section 4.2.2, Page 5 |
| Vault secret management | Section 4.2.2, Page 5 |
| Qwen 7B GGUF | Section 4.0, Page 3 |
| Nemotron-3-22B | Section 4.0, Page 3 |
| OpenAI gpt-4-turbo | Section 4.0, Page 3 |
| Orchestrator Model (1B) | Section 7.0, Page 6 |
| Statistical Routing | Section 7.0, Page 6 |