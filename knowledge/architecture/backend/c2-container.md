# llm-switch Container Architecture (C2)

This diagram illustrates the container-level architecture of the llm-switch system, showing the primary containers and their interactions with external AI applications. The system is deployed in a Nomad cluster with Consul for service discovery and Vault for secret management.

```mermaid
C4Container
    title llm-switch Container Diagram
    System_Boundary(boundary, "llm-switch") {
        Container(llm_switch, "llm-switch", "Go, bifrost", "Main application handling API requests and routing")
        Container(consul_agent, "consul-agent", "Consul", "Consul agent for service discovery")
        Container(vault_agent, "vault-server", "Vault Agent", "Vault agent for secret retrieval")
        Container(nomad_client, "nomad-client", "Nomad Client", "Nomad client agent for node information")
        Container(qwen_local, "qwen-local", "vLLM, Qwen 35B", "Local Qwen model service")
        Container(nemotron_local, "nemotron-local", "vLLM, Nemotron 3 Super", "Local Nemotron model service")
        Container(frontier_gateway, "frontier-api-gateway", "Go, bifrost", "Gateway to frontier API services")
    }
    Person_Ext(ai_app, "AI Application", "External AI application using llm-switch")
    Rel(ai_app, llm_switch, "API Request (OpenAI/Anthropic-compatible)", "HTTP/1.1")
    Rel(llm_switch, consul_agent, "Service Discovery", "HTTP/1.1")
    Rel(llm_switch, vault_agent, "Secret Retrieval", "HTTP/1.1")
    Rel(llm_switch, nomad_client, "Node Metadata", "HTTP/1.1")
    Rel(llm_switch, qwen_local, "Inference Request", "HTTP/1.1")
    Rel(llm_switch, nemotron_local, "Inference Request", "HTTP/1.1")
    Rel(llm_switch, frontier_gateway, "Inference Request", "HTTP/1.1")
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Relationship Descriptions

- **AI Application → llm-switch**: External AI applications send API requests (OpenAI/Anthropic-compatible) to the llm-switch container via HTTP/1.1.
- **llm-switch → Consul Agent**: The llm-switch container queries the Consul agent for service discovery to locate available model services.
- **llm-switch → Vault Agent**: The llm-switch container retrieves secrets (e.g., API keys for frontier models) from the Vault agent.
- **llm-switch → Nomad Client**: The llm-switch container gathers node metadata and health information from the Nomad client agent.
- **llm-switch → Qwen Local**: The llm-switch container sends inference requests to the local Qwen 35B model service via HTTP/1.1.
- **llm-switch → Nemotron Local**: The llm-switch container sends inference requests to the local Nemotron 3 Super model service via HTTP/1.1.
- **llm-switch → Frontier API Gateway**: The llm-switch container forwards requests requiring frontier model capabilities to the frontier API gateway container.