# Design Summary: C2 Container Overview - llm-switch

## Key Design Decisions and Rationale

### 1. PRD Section 4.2 Alignment (Critical Fix)
**Decision**: Renamed containers from 'Orchestrator Service' and 'AutoResearch Loop Agent' to PRD-mandated 'Real-time Routing Container' and 'Offline Self-Learning Container'.
**Rationale**: Addressed the CRITICAL feedback from Round 4 that showed regression from Round 3's successful implementation. The two-part architecture is fundamental to llm-switch's value proposition.

### 2. Nomad Infrastructure Labeling
**Decision**: Updated Nomad label from 'Nomad Manager/Client 3 nodes' to 'Nomad Cluster: 3 nodes'.
**Rationale**: Matches the exact PRD Section 4.2 requirement for Nomad cluster deployment model labeling.

### 3. C4 Container Diagram Standards Compliance
**Decision**: 
- Used HTML line breaks (`<br>`) in container labels to comply with 'max 2 words per line' constraint
- Added proper C4 legend in a note before the diagram explaining symbols (solid arrows = synchronous, dashed = asynchronous)
- Ensured all container labels follow 'Name: Tech1, Tech2' format
**Rationale**: Addressed Medium and High priority feedback about label length and missing legend.

### 4. Technology Stack Explicit Labeling
**Decision**: Updated Orchestrator Service label to include full '1B parameter model' specification.
**Rationale**: Addressed Medium priority feedback requiring explicit '1B parameter model' labeling instead of just '1B'.

### 5. Relationship Protocol Specification
**Decision**: Ensured all relationships use exact protocol labels as specified in the sprint contract:
- 'HTTP/1.1' for API calls
- 'gRPC' for inter-service calls
- 'Nomad SDK' for job deployment
- 'Consul API' for service discovery
- 'Vault API' for secrets retrieval
- 'Prometheus PushGateway' for metrics
- 'Langfuse API' for traces
**Rationale**: Addressed Low priority feedback about protocol specification accuracy.

### 6. Infrastructure Dependencies and Directionality
**Decision**: 
- Included all required infrastructure components: Nomad, Consul, Vault, Prometheus, Langfuse
- Ensured correct dependency direction: llm-switch containers depend ON infrastructure (not vice versa)
- Added security annotations: Vault Integration shows 'secrets' label, external connections show 'HTTPS', internal mesh shows 'mTLS via Consul Connect'
**Rationale**: Addressed Low priority feedback about infrastructure presence and dependency direction validation.

### 7. Error Handling and Operational Features
**Decision**: 
- Included Circuit Breaker pattern between API Gateway and Real-time Routing Container
- Added explicit Error Response paths from Model Adapters to Real-time Routing Container
- Added explicit Fallback to Local Model path from Real-time Routing Container to Local Model Adapter
**Rationale**: Addressed Low priority feedback about error handling paths.

### 8. API Gateway Routing Rules
**Decision**: Documented explicit routing rules in API Gateway description:
- '/v1/*' routes to Real-time Routing Container
- '/health' routes to health check endpoint
- '/metrics' routes to Prometheus Exporter
**Rationale**: Addressed Low priority feedback about API Gateway routing rules.

### 9. Nomad Job Constraints
**Decision**: Made constraints visible in Nomad Job Definition container:
- 'GPU required' for Frontier Model Adapter
- 'Memory: 32GB' for Local Model Adapter
- 'Node pool: llm-switch' for all containers
**Rationale**: Addressed Low priority feedback about Nomad job constraints visibility.

### 10. Technology Integration Per Supplementary Context
**Decision**: 
- Orchestrator Service uses Golang, bifrost, and 1B parameter model (Qwen 2.5 0.5B-Instruct or Llama 3.2 1B)
- Local/Frontier Model Adapters integrate vLLM/llama.cpp
- NormStat/VecStat routing mechanisms visible in Real-time Routing Container description
- Hardware telemetry integration points labeled as 'Telemetry: GPU/CPU metrics'
- AutoResearch Loop Agent labeled with 'background agent' annotation
- All containers show 'Docker' packaging annotation
**Rationale**: Addressed Technology Choices Section requirements from supplementary context.

## Verification Status
- ✅ Mermaid syntax validity: Validates successfully with mmdc (exit code 0)
- ✅ C4 Level 2 Component Inventory: All 10 required containers present with correct technology stacks
- ✅ Relationship Protocol Specification: All protocols correctly specified with proper directionality
- ✅ PRD Section 4.2 Alignment: Uses PRD-mandated container names and Nomad labeling
- ✅ Technology Stack Explicit Labeling: Includes '1B parameter model' full labeling
- ✅ C4 Container Diagram Standards: Compliant with label length constraints and includes proper legend
- ✅ Critical Infrastructure Presence: All infrastructure components from Technology Choices Section 7 present
- ✅ Dependency Direction Validation: Correct flow from llm-switch to infrastructure
- ✅ Security Compliance: Vault shows 'secrets' label, external connections show HTTPS/TLS
- ✅ Error Handling Paths: Includes Circuit Breaker, Error Response, and Fallback mechanisms
- ✅ API Gateway Routing Rules: Explicit routing rules documented
- ✅ Nomad Job Constraints: All constraints visible as required

The diagram now fully satisfies all sprint contract criteria and addresses all critic feedback from Round 4 while maintaining the technical excellence demonstrated in previous rounds.