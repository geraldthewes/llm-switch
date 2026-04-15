# Edge Functions Container Architecture (C2)

This diagram shows the container-level architecture for the llm-switch edge functions component, which handles API requests at the network edge with authentication, rate limiting, and request routing to the core llm-switch service.

## Container Diagram

```mermaid
C4Container
    title Edge Functions Container Architecture for llm-switch
    System_Boundary(edge_functions, "Edge Functions Service") {
        Container(api_gateway, "API Gateway<br>Golang, bifrost, Docker", "Handles incoming API requests", "Entry point for all edge functions")
        Container(auth_container, "Auth Container<br>Golang, Vault API", "API key and JWT validation", "Validates X-API-Key headers and JWT tokens")
        Container(rate_limiter, "Rate Limiter<br>Redis, Golang", "Request rate limiting", "Enforces 100 req/min per API key")
        Container(router, "Request Router<br>Golang, bifrost", "Intelligent request routing", "Routes requests to appropriate backend services")
        Container(observability, "Observability<br>Golang, Prometheus client", "Metrics and logging", "Exports Prometheus metrics and structured logs")
        ContainerDb(cache, "Request Cache<br>Redis", "Frequently accessed data", "Caches routing decisions and model metadata")
    }
    System_Ext(consul, "Consul", "Service discovery and configuration", "Service mesh and KV store for configuration")
    System_Ext(vault, "Vault", "Secret management", "Secure storage for API keys and JWT signing keys")
    System_Ext(core_llm_switch, "Core llm-switch Service", "Main LLM routing logic", "Deployed as Nomad job in cluster")
    System_Ext(langfuse, "Langfuse", "Trace collection", "Async trace accumulation for observability")
    
    Rel(api_gateway, auth_container, "Forwards request for validation", "HTTP/1.1 TLS")
    Rel(auth_container, vault, "Retrieves API key validation data from secret/data/edge/api", "Vault API TLS")
    Rel(auth_container, consul, "Checks IP whitelist", "Consul API")
    Rel(api_gateway, rate_limiter, "Checks rate limit after auth", "HTTP/1.1")
    Rel(rate_limiter, cache, "Reads/writes rate limit counters", "Redis protocol")
    Rel(api_gateway, router, "Forwards validated request for routing", "HTTP/1.1")
    Rel(router, core_llm_switch, "Routes LLM request to core service", "HTTP/1.1")
    Rel(router, cache, "Reads routing decision cache", "Redis protocol")
    Rel(observability, langfuse, "Exports trace data", "Langfuse API")
    System_Ext(prometheus, "Prometheus", "Metrics collection and storage", "Time series database")
    Rel(observability, prometheus, "Exports metrics", "Prometheus exposition format")
    Rel_Back(core_llm_switch, router, "Returns LLM response")
    Rel_Back(router, api_gateway, "Returns routed response")
    Rel_Back(api_gateway, auth_container, "Returns validation result")
    Rel_Back(rate_limiter, api_gateway, "Returns rate limit status")
    Rel_Back(observability, api_gateway, "Logs request details")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Relationship Descriptions

- **API Gateway to Auth Container**: Forwards incoming requests for authentication validation using HTTP/1.1 with TLS encryption
- **Auth Container to Vault**: Retrieves API key validation data and JWT signing keys using Vault API over TLS
- **Auth Container to Consul**: Checks IP address against whitelist configurations stored in Consul KV store
- **API Gateway to Rate Limiter**: Sends authenticated requests for rate limit checking using HTTP/1.1
- **Rate Limiter to Cache**: Reads and writes request counters for rate limiting using Redis protocol
- **API Gateway to Router**: Forwards validated requests to the intelligent routing component using HTTP/1.1
- **Router to Core llm-switch**: Routes LLM requests to the main llm-switch service running as a Nomad job using HTTP/1.1
- **Router to Cache**: Reads cached routing decisions and model metadata from Redis to improve routing performance
- **Observability to Langfuse**: Asynchronously exports trace data for request/response pairs and user feedback
- **Observability to Prometheus**: Exposes metrics in Prometheus exposition format for scraping
- **Core llm-switch to Router**: Returns LLM responses back to the router for forwarding to client
- **Router to API Gateway**: Returns routed responses to the API gateway for final delivery to client
- **API Gateway to Auth Container**: Returns authentication validation results (success/failure)
- **Rate Limiter to API Gateway**: Returns current rate limit status and remaining quota information
- **Observability to API Gateway**: Logs request details including timing, routing decisions, and outcomes for debugging

## Key Architectural Decisions

1. **Authentication Flow**: Uses X-API-Key header validation against Vault-stored keys at exact path `secret/data/edge/api` with JWT token extraction and expiration validation (returns 401 for expired/invalid tokens)
2. **Rate Limiting**: Implements token bucket algorithm with 100 requests/minute per API key using Redis backend
3. **Observability**: Provides Prometheus metrics endpoint (/metrics) and health check endpoint (/health) with structured JSON logging
4. **Security**: Implements mutual TLS for service-to-service communication, IP whitelisting, and CORS restrictions
5. **Error Handling**: Returns standardized error responses:
    - HTTP 401: `{"error":"unauthorized","message":"Invalid or expired token","timestamp":"RFC3339"}`
    - HTTP 403: `{"error":"forbidden","message":"Insufficient permissions","timestamp":"RFC3339"}`
    - HTTP 429: `{"error":"rate_limit_exceeded","message":"Rate limit exceeded","timestamp":"RFC3339"}`
    - HTTP 500: `{"error":"internal_error","message":"Internal server error","timestamp":"RFC3339"}`
    With exponential backoff retry logic (base=100ms, max=5s, maxAttempts=3)
6. **Performance**: Maintains <500ms P99 latency through connection pooling (max_connections=100, timeout=30s) and caching
7. **Scalability**: Designed for horizontal scaling with round-robin load balancing and autoscaling at 80% CPU threshold
8. **Resilience**: Implements circuit breaker pattern with 30s timeout and graceful degradation when dependencies are unavailable