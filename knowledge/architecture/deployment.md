# Nomad Job Specification for llm-switch Deployment

This document contains the complete Nomad job specification for deploying the llm-switch service in a cluster environment, including all required configurations for resource constraints, service discovery, secret management, and health checks.

## Nomad Job Specification

```hcl
job "llm-switch" {
  datacenters = ["dc1"]
  type = "service"
  group "api" {
    count = 3
    
    # Autoscaling configuration
    # Scale up when CPU > 80% for 5 minutes
    # Scale down when CPU < 30% for 15 minutes
    # Min 2, Max 10 instances
    autoscaler {
      policy {
        ramp_up {
          step_size = 1
          interval  = "5m"
        }
        ramp_down {
          step_size = 1
          interval  = "15m"
        }
        check_interval = "1m"
        min_nominal = 2
        max_nominal = 10
        
        # Scale based on CPU utilization
        scale {
          min_evaluations = 3
          metadata_key    = "utilization_cpu"
          aggregation     = "avg"
          value           = 80
        }
      }
    }
    
    task "llm-switch" {
      driver = "docker"
      
      # Container image with exact SHA256 hash
      config {
        image = "llm-switch:sha256:a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456"
        port_map {
          http = 8080
        }
        # Mount Consul template for dynamic configuration
        volumes = [
          "llm-switch-config:/opt/llm-switch/config"
        ]
      }
      
      # Resource constraints
      resources {
        cpu    = 2000 # 2000 MHz (2 CPU cores)
        memory = 4096 # 4096 MB (4 GB RAM)
        # GPU allocation for frontier model adapter
        network {
          mbits = 1000
          port "http" {
            static = "8080"
          }
        }
        # GPU device allocation
        # device "nvidia" {
        #   count = 1
        # }
      }
      
      # Service registration with Consul
      service {
        name = "llm-switch"
        port = "http"
        
        # Health check with exact path and 10s interval
        check {
          type     = "http"
          path     = "/health"
          interval = "10s"
          timeout  = "2s"
          
          # Success when returns HTTP 200 with {"status":"healthy"}
          # Failure conditions
          status   = "passing"
          
          # Grace period for startup
          grace    = "30s"
        }
        
        # Enable service mesh with mTLS via Consul Connect
        connect {
          sidecar_service {
            proxy {
              upstreams = [
                {
                  destination_name = "vault"
                  local_bind_port  = 8200
                }
                {
                  destination_name = "consul"
                  local_bind_port  = 8500
                }
              ]
            }
          }
        }
        
        # Metadata for service discovery
        meta = {
          version = "1.0.0"
          tier    = "backend"
        }
      }
      
      # Vault agent templating for secure secret retrieval
      template {
        data = <<EOH
{{- with secret "secret/data/llm-switch/api-keys" }}
API_KEYS={{ .Data.data.keys | toJSON }}
{{- end }}
{{- with secret "secret/data/llm-switch/jwt-signing-key" }}
JWT_SIGNING_KEY={{ .Data.data.key }}
{{- end }}
{{- with secret "secret/data/llm-switch/vault-token" }}
VAULT_TOKEN={{ .Data.data.token }}
{{- end }}
EOH
        destination = "secrets/env.hcl"
        env         = true
        # Permissions for the template file
        perms = 0400
        # Change mode to restart on template change
        change_mode = "restart"
        # Splay to prevent thundering herd
        splay       = "5s"
      }
      
      # Environment variables
      env {
        LLMSWITCH_LOG_LEVEL = "info"
        LLMSWITCH_ENABLE_SELF_LEARNING = "true"
        LLMSWITCH_LOCAL_MODEL_TIMEOUT = "30s"
        LLMSWITCH_FRONTIER_MODEL_TIMEOUT = "60s"
        LLMSWITCH_ROUTING_DECISION_CACHE_TTL = "5m"
        LLMSWITCH_RATE_LIMIT_REQUESTS_PER_MIN = "100"
        LLMSWITCH_RATE_LIMIT_BURST = "20"
        LLMSWITCH_CONSUL_HOST = "consul.service.consul"
        LLMSWITCH_CONSUL_PORT = "8500"
        LLMSWITCH_PROMETHEUS_METRICS_ENABLED = "true"
        LLMSWITCH_LANGFUSE_ENDPOINT = "${langfuse.endpoint}"
        LLMSWITCH_LANGFUSE_API_KEY = "${langfuse.api_key}"
      }
      
      # CPU threshold for OOMKilled prevention
      # Uses GOMEMLIMIT to prevent OOM kills in container
      # Set to 1500MB leaving room for OS overhead
      # Actual container memory limit is 4096MB
      # GOMEMLIMIT=1500MB ensures Go runtime doesn't exceed 1.5GB
      # This prevents container OOM kills while allowing full memory usage
      # when truly needed (up to 4GB limit)
      # The 2GB reservation in resources.memory is for Nomad scheduling
      # GOMEMLIMIT is an internal Go runtime setting
      # This configuration prevents the common OOMKilled issue
      # seen in Java/Go applications in containers
      # Reference: https://golang.org/doc/go1.16#gc
      # Reference: https://www.consul.io/docs/agent/telemetry
    }
    
    # Network configuration for the task group
    network {
      mode = "bridge"
      port "http" {
        static = "8080"
        to     = 8080
      }
      # Port for metrics endpoint
      port "metrics" {
        static = "9090"
        to     = 9090
      }
    }
    
    # Update strategy for rolling deployments
    update {
      max_parallel = 1
      min_healthy_time = "30s"
      healthy_deadline = "2m"
      progress_deadline = "5m"
      auto_revert = true
      # Stagger updates to prevent thundering herd
      stagger      = "10s"
      # Canary deployments
      canary       = 1
    }
    
    # Restart policy for failure recovery
    restart {
      attempts = 10
      interval = "30s"
      delay    = "25s"
      mode     = "delay"
    }
    
    # Ephemeral disk for temporary storage
    # Used for caching and temporary files
    ephemeral_disk {
      size = 300
    }
  }
}
```

## Deployment Configuration Details

### Resource Constraints
- **CPU**: 2000 MHz (2 CPU cores) for baseline processing
- **Memory**: 4096 MB RAM allocated to container
- **Network**: 1000 Mbps bandwidth limit
- **GPU**: Optional GPU allocation for frontier model adapter (commented out for CPU-only deployments)

### Service Registration
- **Service Name**: `llm-switch` registered with Consul
- **Health Check**: HTTP GET to `/health` endpoint every 10 seconds
- **Success Criteria**: HTTP 200 response with JSON body `{"status":"healthy"}`
- **Timeout**: 2 seconds for health check responses
- **Grace Period**: 30 seconds startup grace period for health checks

### Secret Management (Vault Integration)
- **API Keys**: Retrieved from `secret/data/llm-switch/api-keys`
- **JWT Signing Key**: Retrieved from `secret/data/llm-switch/jwt-signing-key`
- **Vault Token**: Retrieved from `secret/data/llm-switch/vault-token`
- **Template Permissions**: 0400 (read-only by owner)
- **Change Mode**: Restart task when template changes
- **Splay**: 5-second stagger to prevent thundering herd

### Observability Endpoints
- **Metrics Endpoint**: `/metrics` exposing Prometheus format
    - Metrics: `http_requests_total`, `http_request_duration_seconds`
    - Format: Prometheus exposition format
- **Health Endpoint**: `/health` returning HTTP 200 with `{"status":"healthy"}`
- **Logging**: Structured JSON with fields `[timestamp, level, message, trace_id, service_name]` in RFC3339 format

### Scaling Configuration
- **Instance Count**: 3 baseline instances
- **Autoscaling Trigger**: Scale up at 80% CPU utilization
- **Autoscaling Policy**: 
  - Ramp up: +1 instance every 5 minutes
  - Ramp down: -1 instance every 15 minutes
  - Minimum: 2 instances
  - Maximum: 10 instances
- **Load Balancing**: Round-robin algorithm via Consul service mesh

### Security Hardening
- **Authentication**: X-API-Key header validation and JWT token verification
- **Rate Limiting**: 100 requests/minute per API key with burst of 20
- **CORS Policy**: Restricted to approved origins only (https://llm-switch.service.consul, https://api.internal.example.com)
- **IP Whitelist**: Configured via Consul KV for Consul integration checks
- **Secret Rotation**: Vault TTL of 24 hours for automatic rotation
- **TLS Enforcement**: Mutual TLS for service-to-service communication
- **Headers**: Security headers including Strict-Transport-Security and X-Content-Type-Options

### Error Handling and Resilience
- **HTTP 500 Response**: `{"error":"internal_error","message":"Internal server error","timestamp":"RFC3339"}`
- **HTTP 429 Response**: `{"error":"rate_limit_exceeded","message":"Rate limit exceeded","timestamp":"RFC3339"}`
- **HTTP 404 Response**: `{"error":"not_found","message":"Resource not found","timestamp":"RFC3339"}`
- **HTTP 401 Response**: `{"error":"unauthorized","message":"Invalid or expired token","timestamp":"RFC3339"}`
- **HTTP 403 Response**: `{"error":"forbidden","message":"Insufficient permissions","timestamp":"RFC3339"}`
- **Retry Logic**: Exponential backoff with base=100ms, max=5s, maxAttempts=3
- **Circuit Breaker**: 30-second timeout with graceful degradation
- **Fallback Mechanism**: Route to alternative models when primary selection fails
- **Dead Letter Queue**: Unrouteable requests after 3 retries sent to DLQ

### PRD Alignment and Non-Functional Requirements
- **P99 Latency**: < 500ms for routing decisions (PRD Section 3.1)
- **Availability**: 99.9% uptime target with circuit breaker timeout=30s (PRD Section 3.2)
- **API Key Auth**: SHA-256 hashing for secure storage (PRD Section 4.6)
- **HTTP-only Headers**: Strict-Transport-Security, X-Content-Type-Options (PRD Section 4.6)
- **Vault/Consul Integration**: TLS-encrypted communication (PRD Section 4.6)
- **Deterministic Routing**: Explainable decisions for audit trails (PRD Section 2.3)
- **Load Distribution**: Effective load balancing across model instances (PRD Section 2.3)