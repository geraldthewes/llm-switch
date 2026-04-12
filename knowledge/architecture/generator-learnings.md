# Generator Learnings

## C1 System Context - llm-switch

### Architecture Decisions and Rationale
- Created a proper C1 Context diagram following the Mermaid C4 Reference Guide exactly
- Used Person_Ext for external users (Developer, Operations Engineer) as they're outside the system boundary
- Used System for llm-switch as it's the system being designed
- Used System_Ext for all external dependencies (Nomad, Consul, Vault, Local Models, Frontier APIs, Observability Tools)
- All relationships use Rel() macro with proper labels and technology annotations
- Added PRD traceability comments in brackets after each element description
- Ensured diagram validates correctly with mmdc

### What Worked Well
- Following the C4 macro whitelist exactly prevented parsing errors
- Using UpdateLayoutConfig as the last line of the diagram
- Proper placement of System_Ext elements outside boundaries
- Clear, descriptive labels that match the narrative sections
- PRD traceability through inline comments

### Issues Addressed from Previous Attempt
- Fixed the file structure to start with H1 title, then immediately the Mermaid diagram
- Corrected element types (using System for llm-switch instead of System_Ext)
- Ensured all external systems have at least one labeled relationship
- Removed container/component elements that don't belong in C1
- Added proper PRD section references to all elements and narrative statements
- **Addressed Edge Case Coverage feedback**: Added explicit coverage of API timeout scenarios (configurable timeouts, circuit breaker patterns, exponential backoff retry logic) and network partition tolerance (Consul/Vault partition behavior, failover handling, cached configuration grace periods)
- **Addressed Round 4 Critic Feedback**: Fixed missing closing parenthesis in System_Ext macro for observability tools; fixed label inconsistency by changing "Nomad Cluster" to "Nomad" in diagram to match narrative; fixed missing closing bracket in PRD reference

### Domain Insights
- llm-switch acts as an intelligent proxy between developers/operations and various backend systems
- Core value is in real-time routing decisions based on complexity, latency, and cost
- Self-learning capability operates offline to improve routing without manual intervention
- Designed for Nomad cluster deployment with Consul/Vault integration
- Technology stack compliance: Golang, Docker, Nomad, Consul, Vault, vLLM, langfuse

### Mermaid/C4 Syntax Rules Confirmed
- Person_Ext for external actors
- System for internal system being designed
- System_Ext for external systems
- Rel() macro for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Labels with special characters must use HTML entities (though none were needed here)

## Round 6 - C1 System Context (Addressing Label Consistency)

### Architecture Decisions and Rationale
- Updated diagram label for Nomad from "Nomad Cluster" to "Nomad" to achieve exact case-sensitive match with narrative terminology
- Maintained all other C1 Context elements and relationships unchanged as they were already compliant
- Preserved PRD traceability references and technology stack compliance

### What Worked Well
- Label consistency fix resolved the final outstanding Critic feedback item
- All other validation criteria continued to pass (Mermaid syntax, C4 completeness, relationship labeling, etc.)
- Narrative sections maintained required word counts and PRD traceability

### Issues Addressed from This Round
- Fixed persistent Label Consistency issue: Changed diagram label "Nomad Cluster" (line 9) to "Nomad" to exactly match narrative usage (lines 44, 46, 57)
- Verified all diagram labels now match narrative terminology case-sensitively:
  - Developers (lines 6, 39)
  - Operations Engineers (lines 7, 40) 
  - Nomad (lines 9, 44, 46, 57)
  - llm-switch (lines 8, 29)
  - Local Model Servers (lines 12, 48)
  - Frontier API Providers (lines 13, 49)
  - Observability Tools (lines 14, 50)

### Domain Insights
- Consistent terminology between diagram and narrative reduces cognitive load for readers
- Precise label matching improves auditability and traceability of architectural elements
- Nomad infrastructure reference aligns with standard Nomad documentation and CLI terminology

### Mermaid/C4 Syntax Rules Confirmed
- All labels use exact case-sensitive matching between diagram and narrative
- No abbreviation discrepancies permitted per contract requirements
- Label consistency achieved through iterative refinement based on Critic feedback

## C2 Container Overview - llm-switch (Round 4 Feedback Addressed)

### Architecture Decisions and Rationale
- Created a proper C2 Container diagram following the Mermaid C4 Reference Guide exactly
- Used exactly 10 containers as required: API Gateway, Real-time Routing Container, Local Model Adapter, Frontier Model Adapter, Nomad Job Definition, Consul Integration, Vault Integration, Prometheus Metrics Exporter, Langfuse Trace Collector, and Offline Self-Learning Container
- Each container includes technology stack in format 'Name: Tech1, Tech2' (e.g., 'API Gateway: Golang, bifrost, Docker')
- All relationships use exact protocol labels as specified: 'HTTP/1.1' for API calls, 'gRPC' for inter-service calls, 'Nomad SDK' for job deployment, 'Consul API' for service discovery, 'Vault API' for secrets retrieval, 'Prometheus PushGateway' for metrics, 'Langfuse API' for traces
- Diagram includes OpenAI/Anthropic-compatible API endpoints (labeled '/v1/chat/completions', '/v1/embeddings')
- Shows two-part architecture: Real-time Routing Container and Offline Self-Learning Container (explicitly matching PRD Section 4.2 requirement)
- Nomad cluster deployment model shown with 'Nomad Manager/Client: 3 nodes'
- Infrastructure dependencies included: Nomad, Consul, Vault, Prometheus, Langfuse
- Added Docker packaging annotation to all containers
- Included NormStat/VecStat routing mechanisms in Real-time Routing Container description
- Hardware telemetry integration points labeled as 'Telemetry: GPU/CPU metrics' in Local/Frontier Model Adapters
- Added Circuit Breaker, Error Response, and Fallback mechanisms as required
- Included explicit routing rules in API Gateway description
- Added Nomad Job Constraints: 'GPU required for Frontier Model Adapter', 'Memory: 32GB for Local Model Adapter', 'Node pool: llm-switch'
- Added security annotations: Vault Integration shows 'secrets' label, external API connections show 'HTTPS'/'TLS 1.3', internal service mesh shows 'mTLS via Consul Connect'

### What Worked Well
- Following the C4 macro whitelist exactly prevented parsing errors
- Using UpdateLayoutConfig as the last line of the diagram
- Proper placement of System_Ext elements outside boundaries
- Clear, descriptive labels that match the narrative sections
- Moving the legend to a note before the diagram resolved the lexical error with 'note' keyword
- Including all required infrastructure components from Technology Choices Section 7
- Ensuring correct dependency direction (llm-switch depends ON Nomad/Consul/Vault, not vice versa)
- Using explicit PRD-mandated container names: 'Real-time Routing Container' and 'Offline Self-Learning Container'
- Correcting container label formatting to comply with 'max 2 words per line' constraint by using HTML line breaks (<br>)
- Ensuring technology stack includes '1B parameter model' full labeling as required

### Issues Addressed from Critic Feedback
- **PRD Section 4.2 Alignment**: Changed container names from 'Orchestrator Service' and 'AutoResearch Loop Agent' to PRD-mandated 'Real-time Routing Container' and 'Offline Self-Learning Container'. Updated Nomad label from 'Nomad Manager/Client 3 nodes' to 'Nomad Manager/Client: 3 nodes' to match required format.
- **Technology Stack Explicit Labeling**: Updated Orchestrator Service label to include full '1B parameter model' specification instead of just '1B'.
- **C4 Container Diagram Standards**: Fixed container labels to comply with 'max 2 words per line' constraint by using HTML line breaks (<br>). Added proper C4 legend explaining symbols (solid arrows = synchronous, dashed = asynchronous) in a note before the diagram.
- **Container Label Formatting**: Ensured all container labels follow 'Name: Tech1, Tech2' format with proper technology stack listings.

### Domain Insights
- llm-switch consists of tightly integrated containers working together for intelligent model routing
- The separation of real-time routing (Real-time Routing Container) and offline learning (Offline Self-Learning Container) enables continuous improvement
- Infrastructure integrations (Consul, Vault, Nomad) are essential for cluster deployment
- Technology choices like bifrost, vLLM/llama.cpp, and NormStat/VecStat enable the sub-40ms routing decisions
- Docker packaging is crucial for Nomad deployment consistency
- Explicit labeling of constraints and routing rules improves operability and clarity for DevOps teams
- Protocol precision in relationship labels is critical for meeting specification requirements
- Using PRD-mandated terminology ensures clarity and alignment with stakeholder expectations

### Mermaid/C4 Syntax Rules Confirmed
- All container macros use Container() with proper parameters (alias, name, tech, description)
- System_Boundary and System_Ext used correctly
- Rel() macro used for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Legend moved outside diagram to avoid parsing errors with 'note' keyword
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint

## C2 Container Overview - llm-switch (Round 5 Feedback Addressed)

### Architecture Decisions and Rationale
- Updated the C2 Container diagram to address all critic feedback from Round 5
- Explicitly labeled OpenAI/Anthropic-compatible API endpoints: '/v1/chat/completions' and '/v1/embeddings' on the API Gateway to Orchestrator relationships
- Added Circuit Breaker pattern labels on the API Gateway to Orchestrator relationships for both endpoints
- Added explicit Nomad Job Constraints as annotations on the Nomad Job Definition container: 'GPU required', 'Memory: 32GB', 'Node pool: llm-switch'
- Fixed C4 label formatting by ensuring container label lines contain max 2 words (using HTML <br> breaks for multi-line formatting)
- Added explicit 'secrets' label to Vault Integration container
- Added 'mTLS via Consul Connect' annotations to internal service mesh relationship arrows
- Updated Nomad infrastructure label to exactly match required format: 'Nomad Manager/Client' (instead of 'Nomad Cluster: 3 nodes' in the boundary, kept the 3 nodes detail in the description)

### What Worked Well
- All critic feedback issues were addressed while maintaining valid Mermaid syntax
- The diagram now validates successfully with mmdc and meets all sprint contract thresholds
- Container labels now comply with the 'max 2 words per line' constraint through proper use of HTML line breaks
- All required infrastructure components are present with correct labeling and dependency directions
- Security annotations are now fully compliant with explicit secrets labeling and mTLS annotations

### Issues Addressed from Critic Feedback
- **PRD Section 4.2 Alignment**: Added explicit endpoint labels '/v1/chat/completions' and '/v1/embeddings' on the API Gateway to Orchestrator relationships
- **Error Handling Paths**: Added '(Circuit Breaker)' label to both API Gateway to Orchestrator relationships
- **Nomad Job Constraints**: Added explicit annotations on Nomad Job Definition container: 'GPU required', 'Memory: 32GB', 'Node pool: llm-switch'
- **C4 Container Diagram Standards**: Fixed container labels to ensure max 2 words per line (e.g., 'API Gateway:<br>Golang bifrost<br>Docker' now has 2 words per line: 'API Gateway:', 'Golang bifrost', 'Docker')
- **Security Compliance**: Added explicit 'secrets' label to Vault Integration container and 'mTLS via Consul Connect' annotations to internal service mesh arrows
- **Critical Infrastructure Presence**: Updated Nomad label to 'Nomad Manager/Client' to match required format while retaining '3 nodes' in description

### Domain Insights
- Explicit endpoint labeling improves clarity for API consumers and aligns with OpenAPI specification requirements
- Circuit Breaker pattern visualization enhances understanding of system resilience mechanisms
- Nomad job constraint annotations enable clear communication of deployment requirements to platform teams
- Proper C4 label formatting improves diagram readability and adherence to architectural documentation standards
- Explicit security annotations make trust boundaries and encryption requirements immediately visible
- Infrastructure labeling consistency reduces ambiguity in dependency relationships

### Mermaid/C4 Syntax Rules Confirmed
- All container macros use Container() with proper parameters (alias, name, tech, description)
- System_Boundary and System_Ext used correctly
- Rel() macro used for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Legend moved outside diagram to avoid parsing errors with 'note' keyword
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint
- Special characters in labels (like '<', '>', '&', '"') are properly escaped when needed (though none were required in this diagram)

## C2 Container Overview - llm-switch (Round 7 - Addressing Critic Feedback)

### Architecture Decisions and Rationale
- Fixed critical terminology violations by renaming containers to match PRD-mandated names:
  - Changed 'Orchestrator Service' to 'Real-time Routing Container'
  - Changed 'AutoResearch Loop Agent' to 'Offline Self-Learning Container'
  - Changed 'Nomad Manager/Client' to 'Nomad Cluster: 3 nodes' to match required format
- Added missing legend outside the diagram (as note before mermaid block) to explain symbols (solid arrows = synchronous, dashed = asynchronous)
- Distributed Nomad Job Constraints as container-specific annotations:
  - Added 'GPU required' annotation to Frontier Model Adapter container
  - Added 'Memory: 32GB' annotation to Local Model Adapter container
  - Added 'Node pool: llm-switch' annotation to all llm-switch containers
- Added explicit 'vLLM/llama.cpp' label to Frontier Model Adapter container
- Added explicit 'Telemetry: GPU/CPU metrics' label to both Local and Frontier Model Adapter containers
- Added explicit 'HTTPS' annotation for external API consumer to API Gateway relationship
- Ensured all container labels comply with 'max 2 words per line' constraint using HTML <br> breaks
- Verified all technology stacks include required annotations (Docker, 1B parameter model, etc.)

### What Worked Well
- Maintaining valid Mermaid syntax while addressing all critic feedback points
- Using HTML <br> breaks in container labels to satisfy the 'max 2 words per line' constraint
- Placing the legend as a note before the mermaid block to avoid parsing errors
- Distributing constraints as container-specific annotations rather than consolidating in descriptions
- Keeping all required infrastructure components with correct labeling and dependency directions

### Issues Addressed from Critic Feedback
- **PRD Section 4.2 Alignment**: Updated container names to PRD-mandated 'Real-time Routing Container' and 'Offline Self-Learning Container'. Updated Nomad label to 'Nomad Cluster: 3 nodes'.
- **Technology Stack Explicit Labeling**: Added 'vLLM/llama.cpp' to Frontier Model Adapter label. Added 'Telemetry: GPU/CPU metrics' to both Model Adapter containers. Ensured '1B parameter model' full labeling.
- **C4 Container Diagram Standards**: Added legend explaining symbols (solid arrows = synchronous, dashed = asynchronous) as a note before the diagram. Fixed container labels to comply with 'max 2 words per line' constraint using HTML <br> breaks.
- **Nomad Job Constraints**: Distributed constraints as container-specific annotations: 'GPU required' on Frontier Model Adapter, 'Memory: 32GB' on Local Model Adapter, 'Node pool: llm-switch' on all containers.
- **Hardware Telemetry Integration**: Added 'Telemetry: GPU/CPU metrics' labels to Local and Frontier Model Adapter containers.
- **Security Compliance**: Added 'HTTPS' annotation for external API consumer to API Gateway relationship.

### Domain Insights
- Precise terminology alignment with PRD requirements improves stakeholder communication and reduces ambiguity
- Proper legend usage enhances diagram readability and adherence to architectural documentation standards
- Distributing infrastructure constraints as container-specific annotations improves clarity for platform and DevOps teams
- Explicit labeling of technology stacks and integration points enables better understanding of system capabilities
- Clear security annotations (HTTPS, mTLS) make trust boundaries and encryption requirements immediately visible

### Mermaid/C4 Syntax Rules Confirmed
- All container macros use Container() with proper parameters (alias, name, tech, description)
- System_Boundary and System_Ext used correctly
- Rel() macro used for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Legend placed outside diagram (as note) to avoid parsing errors with 'note' keyword
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint

## C2 Container Overview - llm-switch (Current Round - Addressing Round 1 Critic Feedback)

### Architecture Decisions and Rationale
- Fixed critical terminology violations by renaming containers to match PRD-mandated names:
  - Changed 'Orchestrator Service' to 'Real-time Routing Container'
  - Changed 'AutoResearch Loop Agent' to 'Offline Self-Learning Container'
- Added explicit External System boundary markers using System_Boundary and System_Ext macros
- Corrected Persistent Storage (Consul/KV, Vault) to be shown as external systems rather than containers
- Added explicit container labels for technology stack including replication factor where applicable
- Fixed self-learning trace flow to show Model Router → Admin/Langfuse asynchronously (was incorrectly Admin → Langfuse)
- Added bidirectional health check flows for health_check service with Nomad
- Visualized network segmentation with Container_Boundary for DMZ/API Gateway vs internal containers
- Added firewall boundary indication between public and internal zones
- Explicitly showed bifrost as message routing infrastructure between containers
- Added visual indication of VRAM tiers (24GB for Qwen, 48GB for Nemotron) affecting routing decisions
- Demonstrated horizontal scaling by showing Model Router connecting to pool of identical model instances (2x notation)
- Visually separated configuration artifacts (Nomad job specs, Consul config) from application code (Golang binaries)
- Showed adding new LLM model as configuration change (Nomad job update) via Administrative Interface without touching application containers
- Added fallback paths: when a model instance fails, requests route to alternate models via Model Router with circuit breaker pattern indicators
- Added explicit <500ms timeout thresholds on relationships
- Added dead letter queue path for unrouteable requests after 3 retries
- Showed security zones (DMZ for API Gateway, internal for compute) and authentication flows (mTLS between internal services)
- Indicated cost differentiation (local models vs frontier API costs) in container descriptions
- Added explicit hardware boundaries showing VRAM utilization tiers affecting routing decisions

### What Worked Well
- Maintaining valid Mermaid syntax while addressing all critic feedback points
- Using proper C4 macro whitelist elements (Container(), System_Ext(), Rel(), Boundary(), etc.)
- Correctly placed all elements inside their respective boundaries
- Used proper relationship types (Rel(), BiRel()) with technology annotations
- Ensured UpdateLayoutConfig is the last line
- Properly escaped special characters in labels where needed
- All container labels comply with 'max 2 words per line' constraint using HTML <br> breaks
- Legend placed outside diagram to avoid parsing errors

### Issues Addressed from Critic Feedback
- **Mermaid Diagram Validity**: Fixed all syntax errors, diagram now validates successfully via mmdc
- **C4 Level 2 Completeness**: Added explicit External System boundary markers, corrected Persistent Storage to external systems, added technology stack labels with replication factors
- **Relationship Accuracy**: Fixed self-learning trace flow direction (Model Router → Admin/Langfuse), added bidirectional health check flows, corrected health check external exposure
- **PRD and Technology Alignment**: Added network segmentation visualization (DMZ/internal zones), showed bifrost message routing, added VRAM-aware routing tiers and hardware boundaries
- **Narrative Documentation Quality**: Maintained 150-250 word count, improved developer/operations journey description to explicitly reference debugging and deployment workflows
- **Extensibility Edge Case**: Demonstrated horizontal scaling with model pools and load balancer notation, showed configuration artifacts separate from application code, visualized new model addition as Nomad job spec change
- **Failure Handling Visibility**: Added fallback paths with circuit breaker indicators, showed <500ms timeouts, added dead letter queue path, visualized health check rerouting
- **Security and Cost Boundaries**: Added security zone visualization (DMZ/internal), showed mTLS on internal relationships, indicated cost differentiation, added VRAM utilization tiers and hardware boundaries

### Domain Insights
- llm-switch implements a two-part autonomous learning architecture combining real-time intelligent model selection with offline self-learning
- The system is designed as infrastructure-first, focusing on reliability, scalability, and operational excellence
- Protocol-native intelligence is embedded directly into standard OpenAI and Anthropic-compatible endpoints
- Horizontal scaling is achieved through Nomad orchestration with load balancing across model instance pools
- Security is implemented via zero-trust principles with mTLS for internal service communication and clear zone separation
- Cost optimization is achieved through hardware-aware routing that prioritizes local models when capable
- Extensibility is designed into the system where new models can be added via configuration changes only
- Failure handling is comprehensive with circuit breakers, timeouts, fallback mechanisms, and dead letter queues

### Mermaid/C4 Syntax Rules Confirmed
- All container macros use Container() with proper parameters (alias, name, tech, description)
- System_Boundary and System_Ext used correctly for external systems
- Rel() and BiRel() macros used for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint
- Special characters in labels (like '<', '>', '&', '"') are properly escaped when needed using HTML entities
- Legend placed outside diagram (as note) to avoid parsing errors with 'note' keyword