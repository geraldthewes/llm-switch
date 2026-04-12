# C1 System Context: llm-switch

llm-switch operates as an intelligent LLM proxy within Gerald's Nomad cluster infrastructure, acting as a "token factory" that optimizes model selection between local and frontier LLMs. Developers integrate applications by simply redirecting API endpoints to llm-switch, eliminating manual model selection complexity. Operations engineers deploy and maintain the system through Nomad job specifications, leveraging Consul for service discovery and Vault for secret management. The system provides real-time routing based on task complexity while continuously improving through overnight self-learning processes that analyze routing decisions and refine thresholds. All interactions occur via industry-standard OpenAI and Anthropic-compatible APIs, requiring zero code changes to existing applications. The architecture ensures reliability through automatic failover mechanisms and provides observability through Prometheus metrics and health checks for cluster orchestration systems.

```mermaid
C4Context
    title System Context: llm-switch
    Person_Ext(developer, "Developer", "Backend developer integrating AI applications")
    Person_Ext(operations, "Operations", "DevOps engineer managing AI infrastructure")
    System_Ext(nomad, "Nomad orchestrator v1.7.0", "Job scheduler and orchestrator")
    System_Ext(consul, "Consul v1.16.0", "Service discovery and configuration")
    System_Ext(vault, "Vault v1.15.0", "Secret management")
    System_Ext(localQwen, "Qwen 7B GGUF", "Local LLM model")
    System_Ext(localNemotron, "Nemotron-3-22B", "Local LLM model")
    System_Ext(frontierAPI, "OpenAI gpt-4-turbo", "Frontier LLM API")
    
    Boundary(boundaryTrusted, "Trusted Zone (mTLS Required)", dashed) {
        consul
        vault
    }
    
    Boundary(boundaryPublic, "Public Internet\nTLS 1.3 Termination", solid) {
        frontierAPI
    }
    
    System(llmSwitch, "llm-switch", "Intelligent LLM proxy\nGolang 1.21+, Docker 24.0+") {
        System(orchestratorModel, "Orchestrator Model (1B)", "Intent and complexity classification\nFine-tuned Qwen 2.5 0.5B-Instruct")
        System(statisticalRouting, "Statistical Routing", "NormStat/VecStat routing\nTraining-free intent classification")
    }
    
    Rel(developer, llmSwitch, "HTTPS: Developer sends OpenAPI-compatible requests to llm-switch (<50ms>)", "HTTPS")
    Rel(operations, llmSwitch, "HTTPS: Operations configures llm-switch via Nomad job specification (<100ms>)", "HTTPS")
    Rel(llmSwitch, nomad, "HTTPS: llm-switch registers services with Nomad orchestrator (<50ms>)", "HTTPS")
    Rel(llmSwitch, consul, "gRPC: llm-switch queries Consul for service discovery (<10ms>)", "gRPC")
    Rel(llmSwitch, vault, "TLS: llm-switch retrieves API tokens from Vault (<30ms>)", "TLS")
    Rel(llmSwitch, localQwen, "gRPC: llm-switch routes request to Qwen 7B GGUF (<10ms>)", "gRPC")
    Rel(llmSwitch, localNemotron, "gRPC: llm-switch routes request to Nemotron-3-22B (<15ms>)", "gRPC")
    Rel(llmSwitch, frontierAPI, "HTTPS: llm-switch forwards request to OpenAI gpt-4-turbo (<200ms>)", "HTTPS")
    Rel(llmSwitch, frontierAPI, "HTTPS: llm-switch performs failover routing to frontier API on latency threshold breach (>100ms)", "HTTPS", $textStyle="dashed")
    Rel(llmSwitch, vault, "HTTPS: llm-switch performs token refresh with Vault on 401 response (>100ms)", "HTTPS", $textStyle="dashed")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```