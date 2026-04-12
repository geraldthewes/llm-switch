# C2 Container Diagram

llm-switch implements a two-part architecture: real-time intelligent routing selecting optimal models per query, and offline self-learning refining performance overnight. The system is deployed as a Nomad job in a cluster environment with Consul for service discovery and Vault for secrets management.

*Note: Solid arrows = synchronous communication, Dashed arrows = asynchronous communication*

```mermaid
C4Container
    title llm-switch Container Diagram
    System_Boundary(llm-switch, "llm-switch") {
        Container(apiGateway, "API Gateway\nGolang\nbifrost\nDocker", "Exposes /v1/* → Orchestrator\n/health → Health Check\n/metrics → Prometheus Exporter", "")
        Container(orchestrator, "Orchestrator\nService\nGolang\nbifrost\n1B\nparameter\nNormStat/VecStat\nDocker", "Real-time model selection\nHardware telemetry integration\nSub-40ms classification", "")
        Container(localModelAdapter, "Local Model\nAdapter\nGolang\nvLLM\nllama.cpp\nDocker", "Adapts requests to local models\n(Qwen, Nemotron)\nGPU/CPU telemetry", "")
        Container(frontierModelAdapter, "Frontier Model\nAdapter\nGolang\nHTTP\nClient\nDocker", "Adapts requests to frontier APIs\n(OpenAI, Anthropic)\nGPU required", "")
        Container(nomadJob, "Nomad Job\nDefinition\nNomad\nSDK\nDocker", "Constraints: GPU required for Frontier\nMemory: 32GB for Local Model\nNode pool: llm-switch", "")
        Container(consulIntegration, "Consul\nIntegration\nGolang\nConsul\nAPI\nDocker", "Service discovery\nConfiguration retrieval\nmTLS via Consul Connect", "")
        Container(vaultIntegration, "Vault\nIntegration\nGolang\nVault\nAPI\nDocker\nsecrets", "Secrets retrieval\nToken management\nTLS 1.3 encrypted", "")
        Container(prometheusExporter, "Prometheus\nMetrics\nExporter\nGolang\nPrometheus\nClient\nDocker", "Exports metrics at /metrics\nRequest latency, model usage\nError rates tracking", "")
        Container(langfuseTraceCollector, "Langfuse\nTrace\nCollector\nGolang\nLangfuse\nAPI\nDocker", "Asynchronously pushes traces\nRequest/response pairs\nUser feedback signals", "")
        Container(autoResearchAgent, "AutoResearch\nLoop\nAgent\nGolang\nbackground\nagent\nDocker", "Offline self-learning\nAnalyzes traces overnight\nUpdates routing thresholds", "")
    }
    System_Ext(nomad, "Nomad Cluster\n3 nodes", "Orchestration platform")
    System_Ext(consul, "Consul\nService Mesh", "Service discovery and configuration")
    System_Ext(vault, "Vault\nSecrets Store", "Secrets management")
    System_Ext(prometheus, "Prometheus\nServer", "Metrics storage and querying")
    System_Ext(langfuse, "Langfuse\nBackend", "Trace storage and querying")
    System_Ext(localModels, "Local Model\nServers\nQwen, Nemotron, etc.\nGPU\nservers", "Local LLM inference")
    System_Ext(frontierAPIs, "Frontier API\nProviders\nOpenAI, Anthropic, etc.\nHTTPS", "Frontier LLM APIs")

    Rel(apiGateway, orchestrator, "Routes request (with Circuit Breaker)", "HTTP/1.1")
    Rel(orchestrator, localModelAdapter, "Selects local model (primary)", "gRPC")
    Rel(orchestrator, frontierModelAdapter, "Selects frontier model (fallback)", "gRPC")
    Rel(localModelAdapter, orchestrator, "Returns result or error", "HTTP/1.1")
    Rel(frontierModelAdapter, orchestrator, "Returns result or error", "HTTP/1.1")
    Rel(orchestrator, apiGateway, "Returns response", "HTTP/1.1")
    Rel(apiGateway, consulIntegration, "Registers service", "Consul API")
    Rel(consulIntegration, apiGateway, "Returns service info", "Consul API")
    Rel(apiGateway, vaultIntegration, "Retrieves secrets", "Vault API")
    Rel(vaultIntegration, apiGateway, "Returns secrets", "Vault API")
    Rel(apiGateway, prometheusExporter, "Pushes metrics", "Prometheus PushGateway")
    Rel(orchestrator, langfuseTraceCollector, "Pushes traces", "Langfuse API")
    Rel(autoResearchAgent, langfuseTraceCollector, "Reads traces for analysis", "Langfuse API")
    Rel(autoResearchAgent, consulIntegration, "Updates routing thresholds", "Consul API")
    Rel(consulIntegration, orchestrator, "Watches for configuration changes", "Consul API")
    Rel(nomadJob, nomad, "Deploys llm-switch job", "Nomad SDK")
    Rel(localModelAdapter, localModels, "Invokes local model", "HTTP/1.1")
    Rel(frontierModelAdapter, frontierAPIs, "Invokes frontier API", "HTTPS")
    Rel(autoResearchAgent, localModels, "Tests routing hypotheses", "HTTP/1.1")
    Rel(frontierModelAdapter, orchestrator, "Error response or fallback", "HTTP/1.1")
    Rel(localModelAdapter, orchestrator, "Error response or fallback", "HTTP/1.1")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Description

- **API Gateway**: Entry point exposing OpenAI/Anthropic-compatible endpoints (`/v1/chat/completions`, `/v1/embeddings`). Routes requests to the Orchestrator Service with circuit breaker resilience.
- **Orchestrator Service**: Real-time routing engine using a 1B parameter model (Qwen/Llama) via bifrost for sub-40ms complexity classification. Selects between Local and Frontier Model Adapters.
- **Local Model Adapter**: Integrates with local inference servers (vLLM/llama.cpp) for models like Qwen and Nemotron. Handles requests routed to local models.
- **Frontier Model Adapter**: Connects to frontier APIs (OpenAI, Anthropic) via HTTP client for models exceeding local capabilities.
- **Nomad Job Definition**: Encapsulates deployment constraints: GPU requirement for frontier adapters, 32GB memory for local adapters, and node pool affinity. Managed via Nomad SDK.
- **Consul Integration**: Handles service discovery (registering llm-switch) and configuration retrieval (routing thresholds) using Consul API.
- **Vault Integration**: Manages secrets (API keys, tokens) retrieval from Vault using Vault API for secure credential access.
- **Prometheus Metrics Exporter**: Exposes llm-switch metrics (request latency, model usage, error rates) at `/metrics` endpoint for scraping by Prometheus server.
- **Langfuse Trace Collector**: Asynchronously pushes request/response pairs and user feedback to Langfuse backend for trace storage and offline analysis.
- **AutoResearch Loop Agent**: Background process that analyzes Langfuse traces overnight to refine routing decisions, updates Consul configuration, and triggers model retraining.
- **Local Model Servers**: External inference servers (e.g., two RTX 2080s) hosting local models like Qwen and Nemotron.
- **Frontier API Providers**: External services (OpenAPI, Anthropic) accessed via HTTPS for models like GPT-OSS-20B.
- **Nomad Cluster**: Orchestration platform (3-node cluster) deploying the llm-switch job via Nomad SDK.
- **Consul Service Mesh**: Provides service discovery, health checking, and configuration distribution for llm-switch components.
- **Vault Secrets Store**: Centralized secrets management for API keys and tokens used by llm-switch.
- **Prometheus Server**: Metrics storage and querying system scraping llm-switch metrics for alerting and dashboarding.
- **Langfuse Backend**: Trace storage and querying system receiving asynchronous traces from llm-switch for offline analysis.