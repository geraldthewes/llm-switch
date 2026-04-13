# Design Summary for llm-switch Backend Container Architecture (C2)

## Key Decisions and Rationale

### 1. Architecture Boundary Clarification
- **Issue**: Previous diagrams incorrectly placed infrastructure services (Consul, Vault, Nomad) inside a Container_Boundary, implying they were internal to llm-switch
- **Fix**: Modeled all infrastructure services as Container_Ext (external systems) since they are cluster-provided dependencies
- **Rationale**: llm-switch is an application that integrates with existing Nomad/Consul/Vault infrastructure, not a system that contains these services

### 2. Nomad Job Specification Accuracy
- **GPU Resource**: Included exact syntax `gpu = 1` as required for frontier model access
- **Memory Clarification**: Explicitly noted 8192 MB = 8GB (not 8MB) to address previous typo concern
- **Health Checks**: Configured Consul health check at `/health/ready` with 10s interval/3s timeout as specified
- **Vault Integration**: Added `token_renewal = true` and proper ACL policies for long-running service operation

### 3. Technology Stack Compliance
- **Go 1.21+**: Selected for performance and concurrency needed for sub-40ms routing decisions
- **bifrost v0.4.0+**: Chosen for efficient message routing between containers
- **Distroless Base Image**: Minimizes attack surface while ensuring static binary compatibility
- **Hardware-aware Routing**: Integrated vLLM/llama.cpp metrics for VRAM-aware model selection

### 4. Error Handling and Resilience
- **Timeouts**: 30s LLM inference, 5s Consul, 10s Vault operations
- **Retry Logic**: 3 attempts with exponential backoff (1s, 2s, 4s) plus jitter
- **Circuit Breaker**: 5 failures in 30s triggers open state (60s) with half-open recovery testing
- **Dead Letter Queue**: Redis sidecar with PagerDuty alerting (>10 entries in 5-min window)

### 5. Security Implementation
- **Transport**: TLS 1.3 with AES_256_GCM_SHA384 for all external comms
- **Service Mesh**: mTLS via Consul Connect with 24h certificate rotation
- **Secrets**: Vault agent integration with 90-day key rotation and least-privilege policies
- **Authentication**: JWT (15min expiry) and OAuth2 client credentials for service-to-service auth

### 6. Performance Guarantees
- **Latency SLA**: p99 < 200ms at 1000 QPS load
- **Resource Limits**: 8GB container (2GB app heap via GOMEMLIMIT), 4000mc CPU (burst to 6000mc)
- **Connection Limits**: 100 concurrent connections per instance with 80% CPU load shedding
- **GPU Monitoring**: VRAM utilization tracked via hardware telemetry (not hard-limited in Nomad)

### 7. Operational Excellence
- **Observability**: Prometheus metrics endpoint, health checks, structured logging
- **Deployability**: Simple Nomad job specification with sidecar pattern for DLQ
- **Maintainability**: Clear separation of concerns and minimal ongoing intervention
- **Extensibility**: New models added via Nomad job config changes only (no app changes)

## Addressed Critic Feedback from Round 7
- Fixed infrastructure service boundary issue (removed incorrect Container_Boundary)
- Clarified Nomad job memory comment to explicitly state 8GB (value 8192 was already correct)
- Maintained all other high-scoring criteria from previous validation