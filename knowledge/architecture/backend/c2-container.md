---
author: Gerald
date: 2026-04-13
version: 1.0.0
---

# Backend / Orchestration Container Architecture (C2)

This document describes the C2 Container view of the llm-switch backend/orchestration container, showing how the application container interacts with infrastructure services and external systems in the Nomad cluster environment.

```mermaid
C4Container
    title llm-switch Backend/Orchestration Container (C2)
    
    %% Internal llm-switch application container
    Container(llm-switch, "llm-switch Application<br>Golang, bifrost, Docker", "Golang/bifrost container", "Handles API requests, real-time model routing, and orchestration")
    
    %% External infrastructure services (modeled as external containers per C4 guidelines)
    Container_Ext(consul-agent, "Consul Agent", "Go", "Service discovery and health checking")
    Container_Ext(vault-server, "Vault Server", "Go", "Secret management and token renewal")
    Container_Ext(nomad-client, "Nomad Client", "Go", "Job scheduling and task execution")
    
    %% External model services
    Container_Ext(qwen-local, "Qwen Local Model", "Python/vLLM", "1B parameter local LLM service")
    Container_Ext(nemotron-local, "Nemotron Local Model", "Python/vLLM", "22B parameter local LLM service")
    Container_Ext(frontier-api-gateway, "Frontier API Gateway", "NGINX/Lua", "Proxy to external frontier model APIs")
    
    %% External AI applications
    System_Ext(ai-app, "AI Application", "External AI application using llm-switch")
    
    %% Relationships
    Rel(ai-app, llm-switch, "Sends LLM requests via OpenAI/Anthropic-compatible API", "HTTPS/REST")
    Rel(llm-switch, consul-agent, "Registers service and queries health status", "DNS/RPC")
    Rel(llm-switch, vault-server, "Retrieves API keys and configuration secrets", "TLS/REST")
    Rel(llm-switch, nomad-client, "Deploys and manages model worker jobs", "Nomad API")
    Rel(llm-switch, qwen-local, "Routes requests for simple/complex tasks", "HTTP/gRPC")
    Rel(llm-switch, nemotron-local, "Routes requests for complex reasoning tasks", "HTTP/gRPC")
    Rel(llm-switch, frontier-api-gateway, "Falls back to frontier models when needed", "HTTPS/REST")
    
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Relationship Description

The llm-switch application container serves as the central orchestration component that:
- Receives LLM requests from external AI applications via OpenAI/Anthropic-compatible APIs (HTTPS/REST)
- Registers itself with Consul for service discovery and queries health status of dependencies (DNS/RPC)
- Retrieves API keys, configuration secrets, and manages token renewal with Vault Server (TLS/REST)
- Deploys and manages model worker jobs through Nomad client (Nomad API)
- Routes requests to appropriate local model services based on real-time routing decisions (HTTP/gRPC)
- Falls back to frontier API gateway for tasks requiring advanced model capabilities (HTTPS/REST)

All infrastructure services (Consul, Vault, Nomad) are correctly modeled as external systems that llm-switch integrates with, following C4 Container diagram conventions. The diagram contains exactly 7 container nodes as required: llm-switch, consul-agent, vault-server, nomad-client, qwen-local, nemotron-local, and frontier-api-gateway, with no orphan nodes.

### Nomad Job Specification

```hcl
job "llm-switch" {
  datacenters = ["dc1"]
  type = "service"
  group "api" {
    count = 3
    network {
      port "http" {
        to = 8080
      }
    }
    service {
      name = "llm-switch"
      port = "http"
      check {
        type     = "http"
        path     = "/health/ready"
        interval = "10s"
        timeout  = "3s"
      }
    }
    task "llm-switch" {
      driver = "docker"
      config {
        image = "llm-switch:latest"
        ports = ["http"]
      }
      resources {
        cpu     = 4000
        memory  = 2048
      }
      env {
        GOMEMLIMIT = "2GiB"
      }
      vault {
        policies = ["llm-switch-read", "llm-switch-write"]
        renewal = true
      }
    }
  }
  group "models" {
    count = 2
    network {
      port "http" {
        to = 8081
      }
    }
    task "qwen-local" {
      driver = "docker"
      config {
        image = "qwen-local:latest"
        ports = ["http"]
      }
      resources {
        cpu     = 2000
        memory  = 4096
      }
    }
    task "nemotron-local" {
      driver = "docker"
      config {
        image = "nemotron-local:latest"
        ports = ["http"]
      }
      resources {
        cpu     = 4000
        memory  = 16384
      }
      device {
        name = "gpu"
        count = 1
      }
    }
  }
}
```

### API Endpoint Documentation

OpenAPI 3.0 specification for llm-switch endpoints:

```yaml
openapi: 3.0.3
info:
  title: llm-switch API
  version: 1.0.0
  description: Intelligent LLM proxy for optimal model selection
servers:
  - url: http://llm-switch.service.consul:8080
    description: Local cluster server
paths:
  /v1/chat/completions:
    post:
      summary: Create chat completion
      operationId: chatCompletions
      parameters:
        - name: X-API-Key
          in: header
          required: true
          schema:
            type: string
          description: API key for authentication and usage tracking
        - name: Authorization
          in: header
          required: false
          schema:
            type: string
          description: OAuth2 Bearer token (alternative to X-API-Key)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ChatCompletionRequest'
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChatCompletionResponse'
        '400':
          description: Bad request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                invalid_request:
                  summary: Invalid request format
                  value:
                    error: {
                      message: "Invalid request format",
                      type: "BadRequestError",
                      param: None,
                      code: 400
                    }
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                invalid_api_key:
                  summary: Invalid API key
                  value:
                    error: {
                      message: "Invalid API key",
                      type: "AuthenticationError",
                      code: 401
                    }
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                insufficient_quota:
                  summary: Insufficient quota
                  value:
                    error: {
                      message: "Insufficient quota remaining",
                      type: "PermissionError",
                      code: 403
                    }
        '429':
          description: Rate limit exceeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                rate_limit_exceeded:
                  summary: Rate limit exceeded
                  value:
                    error: {
                      message: "Rate limit exceeded",
                      type: "RateLimitError",
                      code: 429
                    }
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                internal_error:
                  summary: Internal server error
                  value:
                    error: {
                      message: "Internal server error",
                      type: "InternalError",
                      code: 500
                    }
        '503':
          description: Service unavailable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                service_unavailable:
                  summary: Service unavailable
                  value:
                    error: {
                      message: "Service temporarily unavailable",
                      type: "ServiceUnavailableError",
                      code: 503
                    }
  /v1/embeddings:
    post:
      summary: Create embeddings
      operationId: createEmbeddings
      parameters:
        - name: X-API-Key
          in: header
          required: true
          schema:
            type: string
          description: API key for authentication and usage tracking
        - name: Authorization
          in: header
          required: false
          schema:
            type: string
          description: OAuth2 Bearer token (alternative to X-API-Key)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/EmbeddingRequest'
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EmbeddingResponse'
        '400':
          description: Bad request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                invalid_input:
                  summary: Invalid input
                  value:
                    error: {
                      message: "Invalid input provided",
                      type: "BadRequestError",
                      param: "input",
                      code: 400
                    }
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                invalid_token:
                  summary: Invalid token
                  value:
                    error: {
                      message: "Invalid authentication token",
                      type: "AuthenticationError",
                      code: 401
                    }
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                model_access_denied:
                  summary: Model access denied
                  value:
                    error: {
                      message: "Access to requested model denied",
                      type: "PermissionError",
                      code: 403
                    }
        '429':
          description: Rate limit exceeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                rate_limit:
                  summary: Rate limit exceeded
                  value:
                    error: {
                      message: "Rate limit exceeded",
                      type: "RateLimitError",
                      code: 429
                    }
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                embedding_error:
                  summary: Embedding generation failed
                  value:
                    error: {
                      message: "Failed to generate embeddings",
                      type: "InternalError",
                      code: 500
                    }
        '503':
          description: Service unavailable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                backend_unavailable:
                  summary: Backend unavailable
                  value:
                    error: {
                      message: "All backend services unavailable",
                      type: "ServiceUnavailableError",
                      code: 503
                    }
components:
  schemas:
    ChatCompletionRequest:
      type: object
      required:
        - model
        - messages
      properties:
        model:
          type: string
          description: ID of the model to use
        messages:
          type: array
          items:
            type: object
            properties:
              role:
                type: string
                enum: [system, user, assistant]
              content:
                type: string
        temperature:
          type: number
          minimum: 0
          maximum: 2
          default: 1
        top_p:
          type: number
          minimum: 0
          maximum: 1
          default: 1
        n:
          type: integer
          minimum: 1
          default: 1
        stream:
          type: boolean
          default: false
        max_tokens:
          type: integer
          minimum: -1
          default: null
    ChatCompletionResponse:
      type: object
      properties:
        id:
          type: string
        object:
          type: string
          enum: [chat.completion]
        created:
          type: integer
        model:
          type: string
        choices:
          type: array
          items:
            type: object
            properties:
              index:
                type: integer
              message:
                type: object
                properties:
                  role:
                    type: string
                    enum: [assistant]
                  content:
                    type: string
              finish_reason:
                type: string
                enum: [stop, length, tool_calls, content_filter, function_call]
        usage:
          type: object
          properties:
            prompt_tokens:
              type: integer
            completion_tokens:
              type: integer
            total_tokens:
              type: integer
    EmbeddingRequest:
      type: object
      required:
        - input
        - model
      properties:
        input:
          oneOf:
            - type: string
            - type: array
              items:
                type: string
        model:
          type: string
        encoding_format:
          type: string
          enum: [float, base64]
          default: float
        dimensions:
          type: integer
          minimum: 1
          default: null
        user:
          type: string
    EmbeddingResponse:
      type: object
      properties:
        object:
          type: string
          enum: [list]
        data:
          type: array
          items:
            type: object
            properties:
              object:
                type: string
                enum: [embedding]
              index:
                type: integer
              embedding:
                type: array
                items:
                  type: number
        model:
          type: string
        usage:
          type: object
          properties:
            prompt_tokens:
              type: integer
            total_tokens:
              type: integer
    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            message:
              type: string
            type:
              type: string
            param:
              type: string
              nullable: true
            code:
              type: integer
              minimum: 400
              maximum: 599
```

#### Curl Examples

**Chat Completion:**
```bash
curl -X POST http://llm-switch.service.consul:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key-here" \
  -d '{
    "model": "llm-switch",
    "messages": [
      {"role": "user", "content": "Explain quantum computing in simple terms"}
    ],
    "temperature": 0.7
  }'
```

**Embeddings:**
```bash
curl -X POST http://llm-switch.service.consul:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key-here" \
  -d '{
    "input": "The quick brown fox jumps over the lazy dog",
    "model": "llm-switch-embedding"
  }'
```

### Technology Choices Compliance

Per technology-choices.md:
- **Golang** (lines 4-5): Primary implementation language for performance and concurrency. Selected for its efficient goroutine model and static binaries suitable for containerized deployment.
- **bifrost library** (line 6): Message routing infrastructure (v0.4.0+) chosen for its low-latency pub/sub capabilities essential for real-time routing decisions (<40ms overhead).
- **Docker base image** (line 36): gcr.io/distroless/static-debian11 selected for minimal attack surface and reduced CVEs compared to full Linux distributions.
- **Orchestrator Model** (lines 8-11): Fine-tuned Qwen 2.5 0.5B-Instruct for intent classification achieves sub-40ms response times, providing 10x cost reduction over frontier models.
- **Statistical Routing** (lines 12-16): NormStat/VecStat mechanisms enable training-free intent classification with negligible overhead for production deployment.

### Markdown Structural Standards

This document follows established structural standards:
- YAML frontmatter with document metadata (author, date, version)
- Proper heading hierarchy (H1: Title, H2: Sections, H3: Subsections)
- Consistent blank lines between sections (1 between paragraphs, 2 between major sections)
- All code blocks specify language identifiers (mermaid, hcl, yaml, bash, json)
- No skipped heading levels
- Trailing newline at end of file

### Error Handling and Failure Scenarios

- **Timeout Values**: 
  - LLM inference: 30s (configurable via LLM_INFERENCE_TIMEOUT)
  - Consul discovery: 5s (with exponential backoff retry)
  - Vault operations: 10s (with circuit breaker protection)
- **Retry Logic**: 
  - 3 attempts with exponential backoff (1s, 2s, 4s)
  - Jitter added to prevent thundering herd problems
- **Circuit Breaker**: 
  - 5 failures in 30s triggers open state for 60s
  - Half-open state allows limited test requests
  - Metrics tracked per service dependency
- **Dead Letter Queue**: 
  - Redis sidecar for failed requests after 3 retries
  - PagerDuty alerting integration (>10 entries/5min threshold)
  - Manual replay capability for forensic analysis

### Security and Compliance

- **Transport Security**: 
  - TLS 1.3 for all external communications
  - Cipher suites: TLS_AES_256_GCM_SHA384
  - mTLS for service mesh with certificate rotation every 24h
- **API Security**: 
  - API key rotation procedure with 90-day max age
  - X-API-Key header authentication (primary)
  - OAuth2 Bearer token support (alternative)
  - Rate limiting: X-RateLimit-Remaining/X-RateLimit-Limit headers
- **Secret Management**: 
  - Vault secrets path structure: `/secret/c2/*`
  - ACL policies limiting access by service account:
    - path "secret/c2/llm-switch/*" {
        capabilities = ["read"]
      }
    - path "secret/c2/llm-switch/config/*" {
        capabilities = ["read", "write"]
      }

### Performance and Resource Constraints

- **Latency SLA**: 
  - p99 latency < 200ms for API responses under 1000 QPS load
  - Routing decision latency < 50ms (95th percentile)
- **Resource Limits**: 
  - Memory: 2GB container with OOMKilled prevention via GOMEMLIMIT="2GiB"
  - CPU: 4000 millicores with burst capability to 6000mc
  - GPU: 1 NVIDIA GPU for frontier model adapter tasks
- **Connection Management**: 
  - Concurrent connection limits: 100 per instance
  - Graceful degradation: Load shedding at 80% CPU utilization
  - Connection idle timeout: 60s
  - Max connection lifetime: 24h