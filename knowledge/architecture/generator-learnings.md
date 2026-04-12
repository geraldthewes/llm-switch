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