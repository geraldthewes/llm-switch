# Generator Learnings - llm-switch C1 Context

## Architecture Decisions Made
- Created C1 System Context diagram showing llm-switch as central intelligent LLM proxy
- Included all required 8 components: llm-switch system, Developer/Persons, Nomad orchestrator, Consul, Vault, Local Models (Qwen 7B GGUF, Nemotron-3-22B), Frontier API (OpenAI gpt-4-turbo)
- Used proper C4 syntax with C4Context block, Person_Ext/System_Ext for external entities, System for internal
- Added security boundaries: dashed for Trusted Zone (Consul/Vault requiring mTLS), solid for Public Internet (frontier APIs)
- Implemented fallback relationships using Rel_Back for latency-based failover and token refresh
- Included specific latency bounds and descriptive relationship labels meeting the ≥25 character requirement
- Specified exact technology versions: Golang 1.21+, Docker 24.0+, Nomad 1.7.0+, Consul 1.16.0+, Vault 1.15.0+
- Explicitly referenced PRD Section 4.2 User Journeys in the narrative as required
- Updated PRD Traceability Matrix to include page numbers in the format 'Section X.Y, Page Z'
- Corrected fallback relationship labels to exactly match required text: 'Failover on latency >100ms' and '401 → Token Refresh'
- Cited supplementary context Section 3.1 for the memory-priority hardware-aware routing strategy
- Used exact wording 'Kubernetes Node Affinity constraints' in the narrative
- Fixed duplicate frontierAPI definition by placing it only inside the Public Internet boundary
- Ensured relationship labels follow format: "Protocol: <Source> performs <Action> on <Destination>" with latency bounds
- Achieved narrative word count of 315 words (within 300±10% range)
- Maintained all 8 required components in the diagram

## What Worked Well with Critic
- Mermaid syntax validated successfully with mmdc
- All required components present per C4 Level 1 completeness criteria
- Relationship labels follow format: "Protocol: <Source> performs <Action> on <Destination>" with latency bounds
- Security boundaries properly delineated with Trusted Zone and Public Internet
- PRD traceability matrix maps components to specific sections with page numbers
- Cluster environment specificity references Prometheus /metrics, Grafana dashboard llm-switch-overview, hardware-aware routing strategy from supplementary context Section 3.1, and Kubernetes Node Affinity constraints
- Fixed duplicate element definition that was causing ambiguity
- Corrected relationship label length to be ≥25 characters
- Used exact term 'memory-priority' from supplementary context
- Changed 'GPU node affinity' reference to 'Kubernetes Node Affinity constraints'

## Issues Addressed from Critic Feedback
- Added explicit reference to PRD Section 4.2 User Journeys in the narrative prose (not just in the matrix)
- Changed fallback relationship labels to exactly match required text: 'Failover on latency >100ms' and '401 → Token Refresh'
- Updated PRD Traceability Matrix to include page numbers (e.g., 'Section 4.2.1, Page 5')
- Added explicit citation to supplementary context Section 3.1 for the memory-priority routing strategy
- Changed 'GPU node affinity' reference to 'Kubernetes Node Affinity constraints' in the narrative

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
- Latency bounds should be in angle brackets like "<50ms>"
- Elements inside a Boundary block must be defined there, not referenced by alias
- Pre-defined elements cannot be moved into a boundary retroactively