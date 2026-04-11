# Generator Learnings - llm-switch C1 Context

## Architecture Decisions Made
- Created C1 System Context diagram showing llm-switch as central intelligent LLM proxy
- Included all required 8 components: llm-switch system, Developer/Persons, Nomad orchestrator, Consul, Vault, Local Models (Qwen 7B GGUF, Nemotron-3-22B), Frontier API (OpenAI gpt-4-turbo)
- Used proper C4 syntax with C4Context block, Person_Ext/System_Ext for external entities, System for internal
- Added security boundaries: dashed for Trusted Zone (Consul/Vault requiring mTLS), solid for Public Internet (frontier APIs)
- Implemented fallback relationships using Rel_Back for latency-based failover and token refresh
- Included specific latency bounds and descriptive relationship labels meeting the ≥25 character requirement
- Specified exact technology versions: Golang 1.21+, Docker 24.0+, Nomad 1.7.0+, Consul 1.16.0+, Vault 1.15.0+

## What Worked Well with Critic
- Mermaid syntax validated successfully with mmdc
- All required components present per C4 Level 1 completeness criteria
- Relationship labels follow format: "Protocol: <Source> performs <Action> on <Destination>" with latency bounds
- Security boundaries properly delineated with Trusted Zone and Public Internet
- PRD traceability matrix maps components to specific sections
- Cluster environment specificity references Prometheus /metrics, Grafana dashboard llm-switch-overview, hardware-aware routing strategy, and Node Affinity

## Issues Addressed from Critic Feedback
- Fixed relationship syntax to use C4 Rel/Rel_Back functions instead of --> arrows
- Added missing Nomad orchestrator component
- Corrected boundary syntax to use Boundary() with proper dashed/solid styling
- Ensured all relationship labels are ≥25 characters with specific format
- Added fallback relationships for local model unavailability and Vault token expiry
- Included exact version numbers for all technologies
- Verified no internal services exposed to Public Internet boundary
- Added cluster environment references to Prometheus endpoint, Grafana dashboard, hardware-aware routing, and Node Affinity

## Domain Insights from PRD
- llm-switch acts as intelligent infrastructure ("token factory") focusing on reliability and operational excellence
- Two-part architecture: real-time routing + offline self-learning system
- Designed for zero-integration-cost compatibility with existing AI applications via OpenAPI/Anthropic-compatible endpoints
- Key user journeys show Developer (Maya) integrating via API change and Operations (Raj) deploying via Nomad
- System reduces frontier model API costs through local model utilization while maintaining <500ms response SLA

## Mermaid/C4 Syntax Rules Confirmed
- C4Context block must be used for System Context diagrams
- External systems/actors use _Ext suffix (Person_Ext, System_Ext)
- Relationship functions: Rel(from, to, "label", "technology") and Rel_Back for reverse
- UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1") required at end
- Boundary() containers accept "dashed" or "solid" parameters
- Do NOT use %%{init}%% directives (GitHub ignores them)
- Use HTTPS/HTTP in relationship labels for technology specification
- Latency bounds should be in angle brackets like "<50ms"