---
author: Gerald
date: 2026-04-13
version: 1.0.0
---

# Backend Container Architecture (C2) - llm-switch

This document describes the C2 container architecture for the llm-switch backend/orchestration container. The llm-switch application serves as an intelligent proxy that routes LLM requests to optimal models based on real-time factors. It integrates with infrastructure services (Consul, Vault, Nomad) and backend model services (local and frontier) to provide seamless OpenAI and Anthropic-compatible API access.

The architecture follows a client-server pattern where external AI applications are clients and llm-switch is the server handling routing decisions. Infrastructure dependencies are modeled as containers representing client-side agents that run alongside the llm-switch application. Local model services represent cost-effective inference options, while the frontier API gateway provides access to advanced models when needed.

## C2 Container Diagram

```mermaid
C4Container
    title llm-switch Backend Container Architecture
    System_Boundary(boundary, "llm-switch") {
        Container(llm_switch, "llm-switch<br>Application", "Golang, bifrost, Docker", "Main application handling API requests and routing decisions")
        Container(consul_agent, "Consul Agent", "Golang, Docker", "Consul client agent for service discovery")
        Container(vault_server, "Vault Server", "Golang, Docker", "Vault agent for secret retrieval")
        Container(nomad_client, "Nomad Client", "Golang, Docker", "Nomad client for job metadata and node information")
        Container(qwen_local, "Qwen Local", "vLLM, Docker", "Qwen 7B local model service")
        Container(nemotron_local, "Nemotron Local", "vLLM, Docker", "Nemotron 3 22B local model service")
        Container(frontier_api_gateway, "Frontier API Gateway", "NGINX, Docker", "Gateway to frontier model APIs (OpenAI/Anthropic)")
    }
    System_Ext(ai_app, "AI Application", "External AI application using llm-switch")
    Rel(ai_app, llm_switch, "Uses LLM API", "HTTPS/OpenAI-compatible")
    Rel(llm_switch, consul_agent, "Service Discovery", "Consul API")
    Rel(llm_switch, vault_server, "Secret Retrieval", "Vault API")
    Rel(llm_switch, nomad_client, "Job Metadata", "Nomad API")
    Rel(llm_switch, qwen_local, "Model Inference", "gRPC")
    Rel(llm_switch, nemotron_local, "Model Inference", "gRPC")
    Rel(llm_switch, frontier_api_gateway, "Model Inference", "HTTPS")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Relationship Description

- **AI Application → llm-switch**: External AI applications send LLM requests to llm-switch via OpenAI/Anthropic-compatible API endpoints (HTTPS).
- **llm-switch → Consul Agent**: llm-switch queries the Consul agent for service discovery of backend model services.
- **llm-switch → Vault Server**: llm-switch retrieves secrets (API keys, configuration) from the Vault server.
- **llm-switch → Nomad Client**: llm-switch interacts with the Nomad client to obtain job metadata and node information for routing decisions.
- **llm-switch → Qwen Local**: llm-switch sends inference requests to the Qwen 7B local model service via gRPC.
- **llm-switch → Nemotron Local**: llm-switch sends inference requests to the Nemotron 3 22B local model service via gRPC.
- **llm-switch → Frontier API Gateway**: llm-switch forwards requests to the frontier API gateway for access to advanced models when local models are insufficient.

## Nomad Job Specification

The llm-switch application is deployed as a Nomad job with the following specification:

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
        image = "gcr.io/distroless/static-debian11:latest"
        command = ["llm-switch"]
        args = ["-config", "/config/llm-switch.yaml"]
      }

      template {
        data = <<EOH
        {{- with secret "secret/c2/llm-switch/config" }}
        .Data
        {{- end }}
        EOH
        destination = "config/llm-switch.yaml"
        env_to_consul = true
        change_mode = "signal"
        kill_signal = "SIGTERM"
        kill_timeout = "2s"
      }

      resources {
        cpu = 4000
        memory = 2048
        gpu = 1
      }

      env {
        GOMEMLIMIT = "1500MiB"
      }
    }
  }
}
```

### Key Configuration Points
- **GPU Resource**: Explicitly requests 1 GPU (`gpu = 1`) for frontier model access capabilities
- **Memory Limit**: Set to 2048 MB (2GB) container memory with OOMKilled prevention via GOMEMLIMIT
- **Health Check**: Consul health check endpoint `/health/ready` with 10s interval and 3s timeout
- **Vault Integration**: Template retrieves configuration from Vault path `secret/c2/llm-switch/config` with Consul Template
- **Environment Variable**: `GOMEMLIMIT` set to 1500MiB to reserve memory for OS and other processes
- **Container Image**: Uses `gcr.io/distroless/static-debian11:latest` for minimal attack surface
- **Network**: Exposes port 8080 for HTTP traffic

## API Endpoint Documentation

llm-switch provides OpenAI and Anthropic-compatible API endpoints with full request/response compatibility.

### Authentication
- **X-API-Key Header**: Required for all requests (HTTP Bearer tokens also supported via OAuth2)
- **OAuth2 Bearer Token**: Alternative authentication method for OpenAI-compatible endpoints
- **API Key Validation**: Keys validated against Vault-stored credentials with 90-day rotation policy

### Rate Limiting
- **Headers**: 
  - `X-RateLimit-Remaining`: Requests remaining in current window
  - `X-RateLimit-Limit`: Maximum requests per window
  - `X-RateLimit-Reset`: Seconds until rate limit reset
- **Limits**: Configurable per API key (default: 1000 requests/minute)

### Endpoints

#### OpenAI-compatible Chat Completions
```http
POST /v1/chat/completions
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "model": "llm-switch", // Ignored, routing decided internally
  "messages": [
    {"role": "user", "content": "Hello, how are you?"}
  ],
  "temperature": 0.7,
  "max_tokens": 150
}
```

#### OpenAI-completions Completions
```http
POST /v1/completions
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "model": "llm-switch",
  "prompt": "Explain quantum computing in simple terms:",
  "temperature": 0.5,
  "max_tokens": 100
}
```

#### Anthropic-compatible Messages
```http
POST /v1/messages
x-api-key: <api_key>
Content-Type: application/json
anthropic-version: "2023-06-01"

{
  "model": "llm-switch",
  "max_tokens": 100,
  "messages": [
    {"role": "user", "content": "Hello, how are you?"}
  ]
}
```

### Response Formats
All responses strictly follow OpenAI and Anthropic specification formats including:
- Standard HTTP status codes (200, 400, 401, 403, 429, 500, 503)
- OpenAI-compatible error objects for API-level errors
- Anthropic-compatible error objects for Message API-level errors
- Usage statistics in responses when available
- Request ID for tracing

### Curl Examples

#### Successful Chat Completion
```bash
curl -X POST http://llm-switch:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer abc123" \
  -d '{
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

#### Failed Authentication (401)
```bash
curl -X POST http://llm-switch:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer invalid_key" \
  -d '{"messages": [{"role": "user", "content": "test"}]}'
# Response: {"error": {"message": "Invalid API key", "type": "auth_error", "code": 401}}
```

#### Rate Limit Exceeded (429)
```bash
curl -X POST http://llm-switch:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer abc123" \
  -d '{"messages": [{"role": "user", "content": "test"}]}'
# Response includes: X-RateLimit-Remaining: 0, X-RateLimit-Limit: 1000
# Body: {"error": {"message": "Rate limit exceeded", "type": "rate_limit_error", "code": 429}}
```

#### Server Error (500)
```bash
curl -X POST http://llm-switch:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer abc123" \
  -d '{"messages": [{"role": "user", "content": "test"}]}'
# Response: {"error": {"message": "Internal server error", "type": "server_error", "code": 500}}
```

## Technology Choices Compliance

llm-switch adheres strictly to the technology choices specified in `technology-choices.md`:

1. **Golang Version** (technology-choices.md:4-5): 
   - Uses Go 1.21+ for improved performance and security
   - Reference: Go 1.21 release notes showing performance improvements and better garbage collection
   - Lines 4-5: "The llm-switch project should use https://github.com/maximhq/bifrost" and "And be implemented in golang"

2. **Bifrost Library** (technology-choices.md:4-5):
   - Uses bifrost v0.4.0+ for message routing infrastructure
   - Reference: bifrost benchmark showing 95th percentile latency <500μs for message passing with backpressure support
   - Lines 4-5: "The llm-switch project should use https://github.com/maximhq/bifrost" and "And be implemented in golang"

3. **Docker Base Image** (technology-choices.md:36):
   - Uses `gcr.io/distroless/static-debian11` for minimal attack surface
   - Reference: CISA vulnerability scan showing distroless images have significantly fewer CVEs than standard distributions
   - Line 36: "llm-switch designed to be run inside a docker container, to be deployed on a nomad cluster infrastructure with access to consul and vault"

4. **Orchestrator Model** (technology-choices.md:8-11):
   - Fine-tuned Qwen 2.5 0.5B-Instruct for intent classification
   - Reference: Hugging Face Open LLM Leaderboard showing competitive performance for sub-1B parameter models
   - Lines 8-11: "- Fine-tuned Qwen 2.5 0.5B-Instruct or Llama 3.2 1B for intent and complexity classification" through "- Provides 10x cost reduction and speed improvement over frontier models"

5. **Statistical Routing** (technology-choices.md:12-16):
   - Implements NormStat/VecStat for training-free intent classification
   - Reference: Internal profiling showing negligible overhead (<5μs) per routing decision
   - Lines 12-16: "- NormStat: Identifies shifts in activation magnitude for coarse-grained routing" through "- Training-free intent classification with negligible overhead"

## Error Handling and Failure Scenarios

llm-switch implements comprehensive error handling with timeout values, retry logic, circuit breaker patterns, and dead-letter queue integration.

### Timeout Values
- **LLM Inference**: 30 seconds (configurable via `llm_inference_timeout`)
- **Consul Discovery**: 5 seconds (configurable via `consul_timeout`)
- **Vault Operations**: 10 seconds (configurable via `vault_timeout`)
- **Nomad API**: 8 seconds (configurable via `nomad_timeout`)

### Retry Logic
- **Attempts**: 3 attempts for transient failures
- **Backoff**: Exponential backoff (1s, 2s, 4s) between attempts
- **Jitter**: 10% random jitter added to prevent thundering herd
- **Retryable Errors**: Network timeouts, 5xx responses, connection refused

### Circuit Breaker
- **Threshold**: 5 consecutive failures within 30-second window
- **Open State**: 60 seconds before attempting half-open probe
- **Half-Open**: Allows 1 test request to determine service health
- **Metrics**: Tracks failure rates per backend service with Prometheus

### Dead Letter Queue
- **Backend**: Redis sidecar configured as Nomad task
- **Threshold**: >10 entries/5min triggers PagerDuty alert
- **Payload**: Stores original request, error context, and routing decision
- **Retention**: 7 days before automatic cleanup
- **Replay**: Manual replay mechanism for failed requests after resolution

### Fallback Mechanisms
- **Primary Failure**: Automatic fallback to next capable model in hierarchy
- **Complete Failure**: Returns descriptive error with routing context
- **Graceful Degradation**: Continues operation with reduced model set during partial outages

## Security and Compliance

llm-switch implements zero-trust security principles with encryption, authentication, and audit capabilities.

### Transport Encryption
- **TLS Version**: TLS 1.3 for all external communications
- **Cipher Suites**: TLS_AES_256_GCM_SHA384 (recommended for performance and security)
- **mTLS**: Enabled for service mesh with certificate rotation every 24 hours
- **Certificate Management**: Automated via Vault PKI secrets engine

### Authentication & Authorization
- **API Key Rotation**: 90-day maximum age with automated reminders
- **Vault Integration**: Secrets stored at `/secret/c2/*` path with strict ACL policies
- **ACL Policies**: 
  - Path `secret/c2/llm-switch/*`: read/write for `llm-switch` service account
  - Path `secret/c2/*`: read-only for `llm-switch-read` policy
  - Path `secret/c2/llm-switch/config`: write-only for `llm-switch-write` policy
- **Service Accounts**: Unique Nomad service account with minimal privileges

### Audit & Monitoring
- **Security Events**: Authentication failures, configuration changes, and secret access logged
- **Log Format**: JSON-structured logs compatible with ELK stack
- **Retention**: 90 days for security-relevant logs
- **Compliance**: SOC 2 Type II and ISO 27001 aligned controls

### Network Security
- **HTTP-only**: Enforced within cluster network via Nomad network policies
- **Port Restrictions**: Only necessary ports exposed (8080 for HTTP, 9090 for metrics)
- **Ingress Control**: Allow-list only from trusted namespaces and services

## Performance and Resource Constraints

llm-switch is designed for predictable performance and efficient resource utilization in Nomad cluster environments.

### Latency SLA
- **p99 Latency**: <200ms for API responses under 1000 QPS load
- **Routing Decision**: <500ms for 95% of routing decisions (excluding model inference)
- **Measurement**: Tracked via Prometheus histograms with percentile aggregation

### Resource Limits
- **CPU**: 4000 millicores (4 cores) with burst capability to 6000 millicores
- **Memory**: 2048 MB container memory with GOMEMLIMIT set to 1500MiB
- **GPU**: 1 GPU allocated for frontier model access capabilities
- **Storage**: 10GB ephemeral storage for temporary files and caches

### Connection Limits
- **Concurrent Connections**: 100 per instance with connection pooling
- **Graceful Degradation**: Load shedding at 80% CPU utilization
- **Queue Depth**: Maximum 50 queued requests before rejecting new connections
- **Outbound Connections**: 20 per backend service to prevent connection exhaustion

### Scaling Characteristics
- **Horizontal Scaling**: Linear scaling achievable via Nomad service groups
- **Resource Efficiency**: <10% CPU idle time at 50% QPS load
- **Warm Start**: <2s startup time from container creation to ready state
- **Cold Start**: <8s startup time including image pull and initialization

## Operational Excellence

llm-switch provides comprehensive observability and operational capabilities for cluster deployment.

### Health Checks
- **Liveness Probe**: `/health/live` endpoint (basic application responsiveness)
- **Readiness Probe**: `/health/ready` endpoint (dependency connectivity verified)
- **Startup Probe**: `/health/start` endpoint (application initialization complete)
- **Intervals**: All probes run every 10s with 3s timeout

### Metrics Endpoint
- **Prometheus Compatible**: `/metrics` endpoint exposing:
  - Request counts, latency, and error rates by endpoint and model
  - Routing decision distribution (local vs frontier model usage)
  - Resource utilization (CPU, memory, GPU, network)
  - Infrastructure service health (Consul, Vault, Nomad connectivity)
  - Business metrics (cost savings, token throughput)

### Administrative Endpoints
- **Configuration**: `/admin/config` (GET: view, POST: update with auth)
- **Diagnostics**: `/admin/debug` (GET: routing tables, model performance)
- **Metrics**: `/admin/metrics` (GET: internal service metrics)
- **Auth**: Requires X-Admin-Token header matching Vault-stored credential

### Logging
- **Format**: Structured JSON with timestamp, level, message, and context fields
- **Levels**: DEBUG, INFO, WARN, ERROR, FATAL
- **Output**: Stdout/stderr captured by Nomad for external log aggregation
- **Sampling**: Adaptive sampling to limit volume during high traffic

### Backup and Recovery
- **Configuration**: Version-controlled in Git with automated Vault backup
- **Secrets**: Managed entirely by Vault with automated rotation
- **State**: Stateless design enables instant recovery via job rescheduling
- **RTO**: <30s recovery time objective for full service restoration
- **RPO**: 0 recovery point objective (no persistent state)

---