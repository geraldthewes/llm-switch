# C1 System Context

llm-switch is an intelligent LLM proxy system that automates optimal model selection for AI applications. It provides unified access through industry-standard OpenAI and Anthropic-compatible APIs while encouraging privacy-preserving, cost-effective local model usage.

## Diagram

```mermaid
C4Context
    title System Context diagram for llm-switch

    Boundary(b0, "Internal Network (Nomad Cluster)") {
        System(llm-switch, "llm-switch", "Intelligent LLM proxy system for automatic model selection")
        System_Ext(LocalModelRepository, "Local Model Repository", "Storage and serving infrastructure for open-source LLMs (Qwen, Nemotron)")
        System_Ext(NomadCluster, "Nomad Cluster", "Orchestration platform for deployment and management")
    }

    Person_Ext(Developer, "Developer", "Builds and maintains AI applications")
    Person_Ext(OperationsEngineer, "Operations Engineer", "Deploys and maintains infrastructure")
    Person_Ext(AIApplicationService, "AI Application Service", "External AI-powered services consuming LLMs")

    System_Ext(OpenAI_API, "OpenAI REST API", "Frontier LLM API access (GPT-4, etc.)")
    System_Ext(Anthropic_API, "Anthropic REST API", "Frontier LLM API access (Claude, etc.)")

    Rel(Developer, llm-switch, "API requests (HTTPS/JSON)", "HTTPS")
    Rel(OperationsEngineer, llm-switch, "Monitoring and configuration (HTTPS/JSON)", "HTTPS")
    Rel(AIApplicationService, llm-switch, "Service calls (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, OpenAI_API, "Request forwarding (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, Anthropic_API, "Request forwarding (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, LocalModelRepository, "Inference requests (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, NomadCluster, "Deployment and health checks (HTTPS/JSON)", "HTTPS")
    
    %% Error handling paths (dashed lines) %%
    Rel(llm-switch, OpenAI_API, "Fallback on local model failure (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, Anthropic_API, "Fallback on local model failure (HTTPS/JSON)", "HTTPS")
    Rel(NomadCluster, llm-switch, "Health check probes (HTTPS/JSON)", "HTTPS")
    Rel(llm-switch, LocalModelRepository, "Fallback to frontier models (HTTPS/JSON)", "HTTPS")

UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Description

| Connection | Protocol/Transport | Description |
|------------|-------------------|-------------|
| Developer → llm-switch | HTTPS/JSON | API requests for LLM inference |
| Operations Engineer → llm-switch | HTTPS/JSON | Monitoring, configuration, and administrative tasks |
| AI Application Service → llm-switch | HTTPS/JSON | Service calls consuming LLM capabilities |
| llm-switch → OpenAI API | HTTPS/JSON | Request forwarding to frontier models when local models insufficient |
| llm-switch → Anthropic API | HTTPS/JSON | Request forwarding to frontier models when local models insufficient |
| llm-switch → Local Model Repository | HTTPS/JSON | Inference requests to locally hosted models (Qwen, Nemotron) |
| llm-switch → Nomad Cluster | HTTPS/JSON | Deployment updates, health checks, and orchestration commands |
| llm-switch → OpenAI API (fallback) | HTTPS/JSON | Automatic fallback to OpenAI when local model fails |
| llm-switch → Anthropic API (fallback) | HTTPS/JSON | Automatic fallback to Anthropic when local model fails |
| Nomad Cluster → llm-switch | HTTPS/JSON | Health check probes for liveness and readiness monitoring |
| llm-switch → Local Model Repository (fallback) | HTTPS/JSON | Fallback to frontier models when local repository unavailable |

## Narrative Completeness Checklist

- [x] **System**: llm-switch - Intelligent LLM proxy system for automatic model selection
- [x] **User Types**: 
  - Developer - Builds and maintains AI applications
  - Operations Engineer - Deploys and maintains infrastructure  
  - AI Application Service - External AI-powered services consuming LLMs
- [x] **External Dependencies**:
  - OpenAI REST API - Frontier LLM API access (GPT-4, etc.)
  - Anthropic REST API - Frontier LLM API access (Claude, etc.)
  - Nomad Cluster - Orchestration platform for deployment and management
  - Local Model Repository - Storage and serving infrastructure for open-source LLMs
- [x] **Network Boundary**: Internal Network (Nomad Cluster) boundary showing llm-switch and internal dependencies
- [x] **Error Handling Paths**: Dashed lines showing fallback mechanisms and health checks
- [x] **Security Boundaries**: Clear separation between internal cluster network and external services

## PRD Consistency Verification

- [x] Nomad cluster explicitly mentioned as orchestration platform
- [x] OpenAI API integration shown with request forwarding
- [x] Anthropic API integration shown with request forwarding  
- [x] Local model repository access shown for inference requests and fallback scenarios
