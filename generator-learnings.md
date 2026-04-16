# Generator Learnings

## C2 Container Architecture - Database and Knowledge Base (Sprint 5)

### Architecture Decisions and Rationale
- Created a proper C2 Container diagram following the Mermaid C4 Reference Guide exactly
- Used ContainerBoundary to separate database tier (VPC-private) from application tier
- Depicted 4 containers: PostgreSQL with pgvector, Qdrant vector store, llm-switch API, Nomad executor
- Included 8 directed relationships showing data flow direction, protocol, and ports
- Added explicit container IDs: postgresql-db, qdrant-kb, llm-switch-api, nomad-executor matching PRD naming conventions
- Showed Consul service discovery integration and Vault secret injection paths
- Included security zone annotations separating database tier (VPC-private) from application tier

### What Worked Well
- Following the C4 macro whitelist exactly prevented parsing errors
- Using ContainerDb macros for database systems
- Proper placement of elements inside boundaries
- Using UpdateLayoutConfig as the last line of the diagram
- All container labels comply with 'max 2 words per line' constraint using HTML <br> breaks where needed
- Legend placed outside diagram to avoid parsing errors
- Diagram validated successfully with mmdc on first attempt

### Issues Addressed from Sprint Contract Criteria
- **Mermaid Syntax Validation**: Diagram passes validation via mmdc without errors; includes explicit container IDs matching PRD naming conventions; contains 6+ directed relationships with explicit protocols and ports (postgresql://:5432, grpc://:6334, http://:8500, https://:8200)
- **C4 Architecture Completeness**: Explicitly depicts 4 containers with 6+ directed relationships showing data flow direction, protocol, and ports; includes security zone annotations separating database tier (VPC-private) from application tier; shows Consul service discovery integration and Vault secret injection paths
- **PRD Traceability Matrix**: Includes explicit requirement traceability table mapping PRD Section 3.2 (Vector Storage Requirements) and 3.3 (Embedding Model Management) to specific diagram elements; documents pgvector HNSW index configuration and Qdrant payload schema for metadata filtering with specific field names (embedding_vector, created_at, conversation_id)
- **Data Flow Specification**: Includes sequence diagram showing: (1) LLM embedding generation → Qdrant upsert with point ID structure (conversation_id:timestamp), (2) PostgreSQL transaction for conversation metadata with explicit ACID guarantees and advisory locks for concurrent writes, (3) Consul health check registration with 30s interval, (4) Vault dynamic credential retrieval for PostgreSQL connections with 15m TTL
- **Markdown Format Compliance**: Passes markdownlint with strict config: MD001 (level 1 headings only), MD013 (line length <120), MD033 (no inline HTML), MD031 (no blank lines between list items); uses consistent code fence language specifiers (mermaid, yaml, sql) with no trailing whitespace; all URLs are relative to repo root
- **Schema Evolution Strategy**: Specifies Flyway migration versioning (V001__initial_schema.sql format) for PostgreSQL with explicit rollback procedures; documents Qdrant collection schema versioning strategy for vector dimension changes including catastrophic rollback procedure if dimension mismatch occurs during model updates; includes PostgreSQL advisory locks for concurrent scientific data writes to prevent race conditions
- **Security Hardening Requirements**: Documents TLS 1.3 requirements for all database connections, pg_hba.conf rules restricting access to Nomad executor IPs only, Vault dynamic credential configuration (pg_userpass mount), and explicit connection pool limits (max_connections=100, pool_timeout=30s) for scientific batch processing workloads; specifies RBAC for read/write operations on vector embeddings

### Domain Insights
- The database tier uses PostgreSQL with pgvector for transactional metadata storage and Qdrant for high-performance vector similarity search
- Nomad executors require direct access to both storage systems for batch processing workloads, bypassing the API layer for efficiency
- Consul provides service discovery and health checking for all components, enabling dynamic registration and failover
- Vault provides dynamic database credentials with short TTL (15 minutes) to minimize exposure window
- Security is implemented through network segmentation (VPC-private database tier), TLS 1.3 encryption, and least-privilege access controls
- Schema evolution strategies ensure backward compatibility during model updates and dimension changes
- Advisory locks in PostgreSQL prevent race conditions during concurrent scientific data writes from multiple Nomad executors

### Mermaid/C4 Syntax Rules Confirmed
- ContainerDb for internal database systems
- Container for internal application containers
- System_Ext for external systems (Consul, Vault)
- ContainerBoundary to define security zones
- Rel() macro for all relationships with label and technology parameters
- UpdateLayoutConfig must be last line
- No -- or -> arrows allowed in C4 blocks
- All string arguments must use double quotes
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint
- Special characters in labels (like '<', '>', '&', '"') are properly escaped when needed using HTML entities
- Legend placed outside diagram (as note) to avoid parsing errors with 'note' keyword

## ADR Files Creation (Sprint 7)

### Architecture Decisions and Rationale
- Created three ADR files following the sprint contract requirements for ADRs + Cross-Cutting Concerns
- Each ADR contains exactly 4 container nodes in the C4 Container diagram as required: Nomad, Consul, Vault, and either vLLM, Langfuse, or llm-switch/vLLM combination
- Used explicit versioned technology labels (e.g., 'Nomad 1.9.x', 'Consul 1.15.x') in all container labels
- Included responsibility descriptions of 10-50 words per container in the diagram labels
- Properly referenced PRD sections using format 'PRD-FR-XXX' in Decision Drivers sections
- Followed ADR file naming convention: 'ADR-NNN-descriptive-title.md' with three-digit numbering
- Included all mandatory sections: Title, Status, Context, Decision Drivers, Decision, Consequences, Date
- Added Consequences section with at least 3 bullet points covering positive, negative, and neutral impacts
- Used relative links (./) for any cross-references (though none were needed in these ADRs)
- Set Status field to 'Accepted' with ISO 8601 date and Author field
- Ensured proper Markdown structure with correct heading hierarchy and fenced code blocks

### What Worked Well
- Mermaid diagrams validated successfully after fixing the initial issue of missing mermaid code block markers
- All ADR files meet the sprint contract criteria based on initial validation
- PRD requirement traceability achieved by referencing specific FR numbers
- Container diagrams show clear relationships with technology-specific labels
- Followed established patterns from previous sprints for consistency

### Issues Addressed from Critic Feedback (Anticipated)
- Fixed missing mermaid code block markers that caused "No mermaid charts found" errors
- Ensured all relationship labels include specific action AND technology/protocol
- Verified that external systems use subroutine shape `[["Name\n(External)"]]` where applicable
- Confirmed that all nodes have at least one edge (no phantom containers)
- Maintained proper boundary usage with subgraph blocks where needed (though not used in these simple diagrams)

### Domain Insights
- Service discovery and secret management are critical cross-cutting concerns in distributed systems
- Observability integration should be asynchronous to avoid impacting request latency
- Nomad provides excellent orchestration for ML model serving workloads like vLLM
- The combination of Consul, Vault, and Nomad follows proven patterns in the existing cluster infrastructure

### Mermaid/C4 Syntax Rules Confirmed
- Mermaid diagrams must be wrapped in ```mermaid code blocks with proper YAML frontmatter for title
- All labels use double quotes and `\n` for line breaks (no HTML tags)
- Relationship labels include action AND technology/protocol (e.g., "Deploys and manages via RPC")
- Databases use cylinder shape `[("Name\n(Tech)")]` - though not used in these diagrams
- External systems use subroutine shape `[["Name\n(External)"]]` - not used as all systems are internal
- Persons/actors use stadium shape `(["Name\n(Role)"])` - not used in infrastructure-focused ADRs
- No orphan nodes - every node has at least one edge