# Database and Knowledge Base C2 Container Architecture

This diagram illustrates the container-level architecture for the database and 
knowledge base components of the llm-switch system. It shows how the llm-switch 
API interacts with persistent storage systems (PostgreSQL with pgvector for 
metadata and Qdrant for vector embeddings), how Nomad executors access these 
stores for batch processing, and the integration with Consul for service 
discovery and Vault for dynamic secret management. Security zones separate the 
private database tier from the application tier.

```mermaid
C4Container
    title Database and Knowledge Base Container Architecture
    
    Container_Boundary(db_tier, "Database Tier (VPC-private)") {
        ContainerDb(postgresql-db, "PostgreSQL with pgvector", "PostgreSQL, pgvector, Flyway", "Stores metadata<br>ACID tx & locks")
        ContainerDb(qdrant-kb, "Qdrant Vector Store", "Qdrant, gRPC", "Stores vectors<br>payload: emb_vector, created_at, conv_id")
    }
    
    Container_Boundary(app_tier, "Application Tier") {
        Container(llm-switch-api, "llm-switch API", "Go, bifrost, Docker", "Handles API requests & routing")
        Container(nomad-executor, "Nomad Executor", "Nomad client, Docker", "Runs batch jobs & tasks")
    }
    
    System_Ext(consul, "Consul", "Service discovery, health checking, key-value store")
    System_Ext(vault, "Vault", "Dynamic secret management, AppRole authentication, pg_userpass mount")
    
    Rel(llm-switch-api, postgresql-db, "Reads/Writes conversation metadata", "postgresql://:5432")
    Rel(llm-switch-api, qdrant-kb, "Vector search/storage", "grpc://:6334")
    Rel(llm-switch-api, nomad-executor, "Job submission/health checks", "http://:8500")
    Rel(nomad-executor, postgresql-db, "Direct DB access for batch jobs", "postgresql://:5432")
    Rel(nomad-executor, qdrant-kb, "Direct vector access for batch jobs", "grpc://:6334")
    Rel(llm-switch-api, consul, "Service registration/health checks", "http://:8500")
    Rel(llm-switch-api, vault, "Dynamic secret retrieval (DB credentials)", "https://:8200")
    Rel(consul, llm-switch-api, "Service discovery queries", "http://:8500")
    Rel(vault, llm-switch-api, "Secret delivery (DB credentials)", "https://:8200")
    
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Relationship Description

The llm-switch API container interacts with both storage systems for real-time operations: 
it reads/writes conversation metadata to PostgreSQL via standard PostgreSQL protocol on port 5432, 
and performs vector search/storage operations with Qdrant via gRPC on port 6334. 
The Nomad executor container accesses the same storage systems for batch processing workloads, 
using identical protocol and port configurations. 
Service discovery and health checking are facilitated through Consul on port 8500, 
with bidirectional communication for registration and queries. 
Vault provides dynamic database credentials to the llm-switch API via HTTPS on port 8200, 
using the AppRole authentication method and pg_userpass mount for PostgreSQL access. 
All database connections enforce TLS 1.3 encryption, and connection pools are limited to 
100 connections with 30-second timeouts to prevent resource exhaustion during batch processing.

### PRD Traceability Matrix

| PRD Section | Requirement | Diagram Element | Description |
|-------------|-------------|-----------------|-------------|
| 3.2 | Vector Storage Requirements | qdrant-kb | Qdrant vector store with HNSW index; payload: embedding_vector (float array), created_at, conversation_id (UUID) |
| 3.2 | Vector Storage Requirements | postgresql-db | PostgreSQL with pgvector for vector storage; supports HNSW and IVFFLAT indexes |
| 3.3 | Embedding Model Management | llm-switch-api | Generates embeddings via local/frontier models; stores in Qdrant with point ID {conversation_id:timestamp} |
| 3.3 | Embedding Model Management | postgresql-db | Stores conversation metadata: model used, token counts, latency for audit |
| 3.2 | ACID Guarantees | postgresql-db | ACID transactions for metadata writes; advisory locks prevent race conditions during concurrent writes |
| 3.2 | Schema Evolution | postgresql-db | Flyway migration versioning (V001__initial_schema.sql); rollback via U001__undo.sql |
| 3.2 | Schema Evolution | qdrant-kb | Collection versioning via alias; dimension mismatch: traffic to old version while new collection builds from snapshot |

### Data Flow Specification

```mermaid
sequenceDiagram
    participant LLM as LLM Switch API
    participant Qdrant as Qdrant Vector Store
    participant Postgres as PostgreSQL with pgvector
    participant Consul as Consul
    participant Vault as Vault
    
    %% Embedding generation and storage
    LLM->>Qdrant: Generate embedding via local/frontier model
    LLM->>Qdrant: Upsert vector with point ID {conversation_id:timestamp}
    Qdrant-->>LLM: Ack with vector ID
    
    %% Conversation metadata storage
    LLM->>Postgres: Begin transaction
    LLM->>Postgres: Insert conversation metadata with advisory lock
    LLM->>Postgres: Commit transaction (ACID guaranteed)
    Postgres-->>LLM: Transaction committed
    
    %% Service discovery
    LLM->>Consul: Register service with health check endpoint (30s interval)
    Consul-->>LLM: Registration ack
    
    %% Dynamic credential retrieval
    LLM->>Vault: Request DB credentials (AppRole role_id, secret_id)
    Vault-->>LLM: Return dynamic DB username/password (15m TTL)
    LLM->>Postgres: Connect using Vault-provided credentials (TLS 1.3)
    Postgres-->>LLM: Connection established
```

### Schema Evolution Strategy

**PostgreSQL with pgvector:** 
- Uses Flyway migration versioning with prefix V001__ for initial schema
- Rollback procedures documented in corresponding U001__undo.sql files
- Advisory locks (pg_try_advisory_xact_lock) prevent race conditions during concurrent scientific data writes
- Connection pool configured: max_connections=100, pool_timeout=30s, validate connections before use

**Qdrant Vector Store:**
- Collection schema versioning via alias pointing to versioned collection names (e.g., llm_switch_vectors_v1)
- Vector dimension changes require new collection creation with updated schema
- Catastrophic rollback procedure: if dimension mismatch occurs during model update, traffic shifts to previous version alias while new collection builds with snapshot restore
- Payload schema enforced: embedding_vector (float array), created_at (timestamp), conversation_id (string)

### Security Hardening Requirements

- **Transport Security:** All database connections enforce TLS 1.3 with certificate verification; Consul and Vault communications use mutual TLS (mTLS) where applicable
- **Network Segmentation:** Database tier resides in VPC-private subnet with security groups restricting ingress to Nomad executor IPs only (port 5432, 6334)
- **Vault Integration:** 
  - Uses AppRole authentication with role_id stored in Vault, secret_id delivered via secure channel
  - Configured pg_userpass mount for dynamic PostgreSQL credential generation
  - Secret TTL set to 15 minutes with automatic renewal
- **Connection Pooling:** 
  - PostgreSQL: max_connections=100, pool_timeout=30s, idle_timeout=60s
  - Qdrant: gRPC connection pool with max_connections=50, keepalive_time=60s
- **Access Control:** 
  - PostgreSQL: pg_hba.conf restrictions allow connections only from Nomad executor CIDR blocks
  - Qdrant: API key protection with read/write separation; llm-switch API has write access, Nomad executor has read-only for batch jobs
  - Vault policies enforce least privilege: llm-switch API can only read database credentials, Nomad executor has no Vault access
- **Audit Logging:** All database access logged via pg_audit; Vault access logged to syslog; Consul service changes monitored via alerts