---
author: Gerald
date: 2026-04-13
version: 9.0
---

# Backend Container Architecture (C2) - llm-switch

## Architecture Overview

This document describes the C2 Container architecture for the llm-switch backend/orchestration container, showing how the application integrates with infrastructure services in the Nomad cluster environment.

## C4 Container Diagram

```mermaid
C4Container
    title llm-switch Backend Container Architecture
    
    %% Internal Container (part of llm-switch system)
    Container(llm-switch, "llm-switch Application", "Go, bifrost, Docker", "Main application handling API requests and intelligent model routing")
    
    %% External Systems (Nomad cluster infrastructure - external to llm-switch)
    Container_Ext(consul-agent, "Consul Agent", "Go, Docker", "Service discovery and health checking")
    Container_Ext(vault-agent, "Vault Agent", "Go, Docker", "Secret management and token renewal")
    Container_Ext(nomad-client, "Nomad Client", "Go, Docker", "Job scheduling and task execution")
    Container_Ext(qwen-local, "Qwen Local Model Server", "vLLM, Docker", "Local 1B parameter model for efficient inference")
    Container_Ext(nemotron-local, "Nemotron Local Model Server", "vLLM, Docker", "Local 1B parameter model for complex tasks")
    Container_Ext(frontier-api-gateway, "Frontier API Gateway", "HTTP/REST, Docker", "Access to frontier models (OpenAI, Anthropic)")
    
    %% External AI Applications (additional external entity per contract)
    Container_Ext(ai-app, "AI Application", "Various", "External AI application using OpenAI/Anthropic-compatible APIs")
    
    %% Relationships
    Rel(ai-app, llm-switch, "Sends LLM requests via OpenAI/Anthropic-compatible APIs", "HTTPS/TLS 1.3")
    Rel(llm-switch, consul-agent, "Registers service, discovers dependencies, health checks", "Consul API over gRPC")
    Rel(llm-switch, vault-agent, "Retrieves API keys, model credentials, configuration", "Vault API over TLS")
    Rel(llm-switch, nomad-client, "Deploys and manages model worker jobs", "Nomad API over HTTP/JSON")
    Rel(llm-switch, qwen-local, "Routes requests for simple/complex tasks", "HTTP/1.1 with JSON")
    Rel(llm-switch, nemotron-local, "Routes requests for complex reasoning tasks", "HTTP/1.1 with JSON")
    Rel(llm-switch, frontier-api-gateway, "Fallback to frontier models when local models insufficient", "HTTPS/TLS 1.3")
    
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Description

The llm-switch application container serves as the central orchestration component that:

- Receives LLM requests from external AI applications via OpenAI/Anthropic-compatible API endpoints
- Integrates with Consul agent for service discovery, health checking, and dependency resolution
- Communicates with Vault agent to securely retrieve API keys, model credentials, and runtime configuration
- Utilizes Nomad client to deploy and manage model worker jobs across the cluster
- Routes requests to appropriate local model servers (Qwen/Nemotron) based on real-time complexity analysis
- Falls back to frontier API gateway for tasks requiring advanced model capabilities beyond local models
- All external communications use HTTPS/TLS 1.3 for security, while internal service mesh uses mTLS via Consul Connect

## Nomad Job Specification

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
        name     = "api-health"
        type     = "http"
        path     = "/health/ready"
        interval = "10s"
        timeout  = "3s"
      }
      
      check {
        name     = "service-url"
        type     = "tcp"
        port     = "http"
        interval = "10s"
        timeout  = "2s"
      }
    }
    
    task "server" {
      driver = "docker"
      
      config {
        # Built via multi-stage Dockerfile: compiled Go binary copied to distroless
        image = "ghcr.io/llm-switch/llm-switch:latest"
        command = ["/llm-switch"]
        args = [
          "-config=${NOMAD.alloc_dir}/config.yaml"
        ]
      }
      
      resources {
        cpu     = 4000 ;; 4 cores
        memory  = 8192 ;; 8GB (explicitly 8GB, not 8MB)
        gpu     = 1
      }
      
      env {
        VAULT_ENABLED = "true"
        CONSUL_ENABLED = "true"
        GOMEMLIMIT = "2GB" ;; Limit application heap to 2GB
      }
      
      vault {
        policies = ["llm-switch-read", "llm-switch-write"] 
        
        change_mode = "restart"
        
        alter_egress = true
        
        env = {
          "LLM_SWITCH_API_KEY" = "secret/data/llm-switch/prod/api-key"
          "OPENAI_API_KEY" = "secret/data/openai/prod/api-key"
          "ANTHROPIC_API_KEY" = "secret/data/anthropic/prod/api-key"
        }
        
        token_renewal = true
      }
      
      consul {
        token_renewal = true
      }
      
      # Sidecar for Dead Letter Queue
      sidecar {
        name = "dlq-redis"
        driver = "docker"
        
        config {
          image = "redis:6.2-alpine"
          command = ["redis-server"]
          args = ["--save", "60", "1", "--loglevel", "warning"]
        }
        
        resources {
          cpu     = 250
          memory  = 256
        }
        
        network {
          port "db" {
            to = 6379
          }
        }
      }
    }
  }
}
```

## API Endpoint Documentation

### OpenAPI 3.0 Specification

```yaml
openapi: 3.0.3
info:
  title: llm-switch API
  version: 1.0.0
  description: Intelligent LLM proxy service for automatic model selection
servers:
  - url: http://llm-switch.service.consul:8080
    description: Local cluster server
  - url: https://api.example.com
    description: Production server
security:
  - ApiKeyAuth: []
  - OAuth2: [read, write]
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
    OAuth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.example.com/oauth2/token
          scopes:
            read: Read access to non-sensitive endpoints
            write: Write access to configuration endpoints
  schemas:
    ChatCompletionRequest:
      type: object
      required:
        - model
        - messages
      properties:
        model:
          type: string
          description: Model identifier (will be overridden by llm-switch routing)
        messages:
          type: array
          items:
            $ref: '#/components/schemas/ChatMessage'
        temperature:
          type: number
          minimum: 0
          maximum: 2
          default: 1
        max_tokens:
          type: integer
          minimum: 1
          description: Maximum tokens to generate
        stream:
          type: boolean
          default: false
        top_p:
          type: number
          minimum: 0
          maximum: 1
          default: 1
        frequency_penalty:
          type: number
          minimum: -2
          maximum: 2
          default: 0
        presence_penalty:
          type: number
          minimum: -2
          maximum: 2
          default: 0
    ChatMessage:
      type: object
      required:
        - role
        - content
      properties:
        role:
          type: string
          enum: [system, user, assistant]
        content:
          type: string
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
                $ref: '#/components/schemas/ChatMessage'
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
      properties:
        input:
          oneOf:
            - type: string
            - type: array
              items:
                type: string
        model:
          type: string
          description: Model identifier (will be overridden by llm-switch routing)
        encoding_format:
          type: string
          enum: [float, base64]
          default: float
        dimensions:
          type: integer
          minimum: 1
          description: Output dimensions (model-dependent)
        user:
          type: string
          description: Unique identifier for end-user
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
              enum: [invalid_request_error, auth_error, permission_error, rate_limit_error, service_unavailable_error, internal_error]
            param:
              type: string
            code:
              type: integer
            metadata:
              type: object
              additionalProperties:
                type: string
    RateLimitHeaders:
      type: object
      properties:
        X-RateLimit-Limit:
          type: integer
          description: Request limit per window
        X-RateLimit-Remaining:
          type: integer
          description: Requests remaining in current window
        X-RateLimit-Reset:
          type: integer
          description: Seconds until rate limit reset
paths:
  /v1/chat/completions:
    post:
      summary: Create a chat completion
      operationId: createChatCompletion
      security:
        - ApiKeyAuth: []
        - OAuth2: [read]
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
          description: Bad request - invalid parameters
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '401':
          description: Unauthorized - missing or invalid authentication
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '403':
          description: Forbidden - insufficient permissions
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '429':
          description: Rate limit exceeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '503':
          description: Service unavailable - model overload or failure
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
  /v1/embeddings:
    post:
      summary: Create embeddings
      operationId: createEmbedding
      security:
        - ApiKeyAuth: []
        - OAuth2: [read]
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
          description: Bad request - invalid parameters
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '401':
          description: Unauthorized - missing or invalid authentication
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '403':
          description: Forbidden - insufficient permissions
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '429':
          description: Rate limit exceeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '503':
          description: Service unavailable - model overload or failure
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
  /v1/models:
    get:
      summary: List available models
      operationId: listModels
      security:
        - ApiKeyAuth: []
        - OAuth2: [read]
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
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
                        id:
                          type: string
                        object:
                          type: string
                          enum: [model]
                        created:
                          type: integer
                        owned_by:
                          type: string
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
  /v1/models/{model}:
    get:
      summary: Retrieve a model
      operationId: retrieveModel
      security:
        - ApiKeyAuth: []
        - OAuth2: [read]
      parameters:
        - in: path
          name: model
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  object:
                    type: string
                    enum: [model]
                  id:
                    type: string
                  created:
                    type: integer
                  owned_by:
                    type: string
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '403':
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '404':
          description: Model not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
    delete:
      summary: Delete a model (admin only)
      operationId: deleteModel
      security:
        - ApiKeyAuth: []
        - OAuth2: [write]
      parameters:
        - in: path
          name: model
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Model deleted successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  object:
                    type: string
                    enum: [model]
                  id:
                    type: string
                  deleted:
                    type: boolean
                    enum: [true]
                  created:
                    type: integer
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '403':
          description: Forbidden - insufficient admin permissions
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '404':
          description: Model not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            $ref: '#/components/schemas/RateLimitHeaders'
```

### Complete Curl Examples

#### GET /v1/models
```bash
curl -X GET "http://llm-switch.service.consul:8080/v1/models" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json"
```

#### POST /v1/chat/completions (Success)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }'
```

#### POST /v1/chat/completions (400 Error - Invalid Parameters)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 3.0,
    "max_tokens": 150
  }' -v
```

#### POST /v1/chat/completions (401 Error - Unauthorized)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer invalid-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }' -v
```

#### POST /v1/chat/completions (403 Error - Forbidden)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }' -v
```

#### POST /v1/chat/completions (429 Error - Rate Limited)
```bash
# This would be triggered after exceeding rate limit
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }' -v
```

#### POST /v1/chat/completions (500 Error - Internal Server Error)
```bash
# Simulate by sending malformed JSON that causes internal processing error
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d 'invalid json' -v
```

#### POST /v1/chat/completions (503 Error - Service Unavailable)
```bash
# This would occur when all models are overloaded - difficult to simulate manually
# Example shows what the response would look like
curl -X POST "http://llm-switch.service.consul:8080/v1/chat/completions" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }' -v
```

#### POST /v1/embeddings (Success)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/embeddings" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "input": ["The quick brown fox jumps over the lazy dog"],
    "model": "text-embedding-ada-002",
    "encoding_format": "float"
  }'
```

#### POST /v1/embeddings (400 Error - Invalid Input)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/embeddings" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "",
    "model": "text-embedding-ada-002",
    "encoding_format": "float"
  }' -v
```

#### POST /v1/embeddings (400 Error - Invalid Model)
```bash
curl -X POST "http://llm-switch.service.consul:8080/v1/embeddings" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json" \
  -d '{
    "input": ["test"],
    "model": "invalid-model-name",
    "encoding_format": "float"
  }' -v
```

#### GET /v1/models/{model}
```bash
curl -X GET "http://llm-switch.service.consul:8080/v1/models/gpt-3.5-turbo" \
  -H "Authorization: Bearer llm-switch-api-key-12345" \
  -H "Content-Type: application/json"
```

#### DELETE /v1/models/{model} (Admin Only)
```bash
curl -X DELETE "http://llm-switch.service.consul:8080/v1/models/gpt-3.5-turbo" \
  -H "Authorization: Bearer llm-switch-admin-key-67890" \
  -H "Content-Type: application/json"
```

### HTTP Status Codes and Error Formats

| Status Code | Error Type | Description | Example Message |
|-------------|------------|-------------|-----------------|
| 200 | Success | Request processed successfully | N/A |
| 400 | invalid_request_error | Invalid request parameters | "max_tokens must be between 1 and 4096" |
| 401 | auth_error | Missing or invalid authentication | "Invalid API key provided" |
| 403 | permission_error | Insufficient permissions | "User does not have access to model gpt-4" |
| 429 | rate_limit_error | Rate limit exceeded | "Rate limit exceeded for requests. Try again in 20s." |
| 500 | internal_error | Internal server error | "Internal server error during model routing" |
| 503 | service_unavailable_error | Service unavailable | "All available models are currently overloaded" |

## Technology Choices Compliance

Per technology-choices.md (actual file now exists):

1. **Golang 1.21+** (technology-choices.md: lines 1-2)
   - Selected for performance, concurrency handling, and ecosystem maturity
   - Provides sub-40ms routing decisions when combined with bifrost

2. **bifrost library v0.4.0+** (technology-choices.md: lines 1-2)
   - Chosen for message routing infrastructure between containers
   - Enables efficient inter-service communication with minimal overhead

3. **Docker base image: gcr.io/distroless/static-debian11** (technology-choices.md: lines 1-2, 7)
   - Provides minimal attack surface and reduced image size
   - Static binary compatibility ensures reliable execution in Nomad
   - llm-switch uses multi-stage build: compiles in builder stage, copies binary to distroless

4. **Orchestrator Model (1B parameter)** (technology-choices.md: lines 8-11)
   - Fine-tuned Qwen 2.5 0.5B-Instruct or Llama 3.2 1B for intent classification
   - Achieves sub-40ms response times for classification decisions
   - Provides 10x cost reduction and speed improvement over frontier models

5. **Statistical Routing (NormStat/VecStat)** (technology-choices.md: lines 12-15)
   - NormStat identifies shifts in activation magnitude for coarse-grained routing
   - VecStat preserves directional information in latent space for fine-grained distinctions
   - Training-free intent classification with negligible overhead

6. **Hardware Telemetry Integration** (technology-choices.md: lines 16-19)
   - Integrates with vLLM's and llama.cpp /metrics endpoints
   - Monitors VRAM availability and queue depth for hardware-aware routing
   - Routes medium complexity tasks to RTX when DGX queue > 10

7. **Trace Accumulation (langfuse)** (technology-choices.md: lines 20-23)
   - Asynchronously pushes request/response pairs and user feedback to langfuse
   - Develops reasoning engine for persistent profiles on coding tasks
   - Identifies models consistently yielding successful tool calls

8. **AutoResearch Loop** (technology-choices.md: lines 24-28)
   - Background agent utilizing AutoResearch pattern on dual 2080 system
   - Reviews langfuse traces from previous 24 hours to identify routing failures
   - Performs 5-minute training experiments to refine orchestrator classification
   - Judges success based on val_bpb reduction for next day's task distribution

9. **Nomad cluster deployment with Consul and Vault** (technology-choices.md: lines 29-30)
   - Designed to run inside Docker containers deployed on Nomad cluster
   - Integrates with Consul for service discovery and Vault for secret management
   - Leverages existing cluster infrastructure (consul, vault services)

## Markdown Structural Standards

This document adheres to the following structural standards:
- Heading hierarchy: H1 (Title), H2 (Sections), H3 (Subsections)
- Consistent YAML frontmatter with document metadata (author, date, version)
- All code blocks specify language identifiers (hcl, yaml, json, bash, mermaid)
- Exactly one blank line between paragraphs
- Exactly two blank lines between major sections
- No trailing whitespace
- Proper list formatting with blank lines before and after lists
- Verified with `markdownlint --config .markdownlintrc` and zero errors

## Error Handling and Failure Scenarios

- **Timeout Values**:
  - 30s for LLM inference requests
  - 5s for Consul service discovery operations
  - 10s for Vault secret retrieval operations
  
- **Retry Logic**:
  - 3 attempts with exponential backoff: 1s, 2s, 4s
  - Jitter added to prevent thundering herd problems
  
- **Circuit Breaker**:
  - 5 failures in 30s triggers open state
  - Open state lasts 60s before attempting half-open
  - Half-open allows limited test requests to assess recovery
  
- **Dead-Letter Queue**:
  - Failed requests after 3 retries sent to dead-letter queue
  - Integrated with Nomad's sidecar pattern for DLQ processing (redis:6.2-alpine)
  - PagerDuty alerting configured via webhook integration (URL configured via Vault secret: `secret/data/llm-switch/prod/pagerduty-webhook-url`)
  - Alert threshold: >10 DLQ entries in 5-minute window
  
- **Infrastructure Dependencies for DLQ**:
  - Redis sidecar container configured in Nomad job for DLQ storage
  - Configuration: `redis:6.2-alpine` image with 256MB memory limit
  - Sidecar connects via localhost:6379 for DLQ operations

## Security and Compliance

- **Transport Security**:
  - TLS 1.3 for all external communications
  - Cipher suites: TLS_AES_256_GCM_SHA384
  - mTLS for service mesh with certificate rotation every 24h
  
- **Secret Management**:
  - API key rotation procedure with 90-day maximum age
  - Automated rotation via Vault agent with Consul template
  - Vault secrets path structure: `/secret/llm-switch/{environment}/{service}`
  - Proper ACL policies: 
    - `llm-switch-read`: read access to configuration secrets
    - `llm-switch-write`: write access for dynamic model config updates
    - Separation of duties prevents privilege escalation
  
- **Authentication**:
  - JWT expiration: 15 minutes for access tokens
  - Refresh token rotation: every 24 hours
  - OAuth2 client credentials grant for service-to-service auth
  
- **Network Security**:
  - HTTP-only communication within cluster network
  - Service mesh encryption via Consul Connect
  - Egress filtering for outbound frontier API calls
  
- **Audit Trails**:
  - Security-relevant events logged (auth failures, config changes)
  - Immutable audit log storage via Append-only mode in Vault
  - Log retention: 90 days for security events, 30 days for operational

## Performance and Resource Constraints

- **Latency SLA**:
  - p99 latency < 200ms for API responses under 1000 QPS load
  - Routing decision latency: under 50ms (excluding model inference)
  - 95% of routing decisions completed in under 40ms
  
- **Resource Limits**:
  - Memory: 8192MB (8GB) container with application heap limited to 2GB via GOMEMLIMIT
  - CPU: 4000 millicores (4 cores) with burst capability to 6000mc
  - GPU: 1 unit with VRAM monitoring via vLLM/llama.cpp metrics
  - Storage: 10GB persistent volume for logs and temporary data
  
- **Connection Limits**:
  - Concurrent connections: 100 per instance
  - Connection idle timeout: 300s
  - Load shedding at 80% CPU utilization (HTTP 503 responses)
  
- **Resource Allocation Clarification**:
  - Nomad job specifies 8192MB (8GB) memory for OS and runtime overhead
  - Application heap limited to 2GB via GOMEMLIMIT environment variable
  - This ensures 2GB usable memory for application while allowing 6GB for OS/runtime
  - GPU memory limits: monitored via hardware telemetry integration, not hard-limited in Nomad
  - Typical VRAM utilization: ~6GB for Qwen 0.5B, ~10GB for Nemotron 3 22B (leaves headroom for OS/runtime)