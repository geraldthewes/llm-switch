# Generator Learnings

## Edge Functions C2 Container - llm-switch (Sprint 6)

### Architecture Decisions and Rationale
- Updated the Vault relationship in edge-functions/c2-container.md to include the exact path `secret/data/edge/api` as required by the sprint contract for edge function authentication flow.
- Added explicit HTTP 401 and 403 error responses in deployment.md to meet the Edge Function Authentication Flow criterion.
- Fixed the CPU unit specification in deployment.md to explicitly state "2000 MHz (2 CPU cores)" to address the minor deduction in the Nomad Job Specification Completeness criterion.
- Corrected the list indentation in edge-functions/c2-container.md to use 4-space indentation for lists as required by the Markdown Syntax and Structure criterion.
- Added exact allowed origins for CORS policy in deployment.md to address the Security Hardening criterion.
- Updated the Error Handling section in edge-functions/c2-container.md to use proper 4-space indentation for sub-bullets to comply with markdownlint.
- Documented JWT extraction algorithm as HS256 in edge-functions/c2-container.md to address the missing cryptographic algorithm requirement.
- Explicitly documented HTTP 401/403 JSON structures with exact field names 'error' and 'message' in edge-functions/c2-container.md.

### What Worked Well
- The Mermaid diagram in edge-functions/c2-container.md validated successfully after the updates.
- All required Nomad job specification elements are present and correctly formatted in deployment.md.
- The authentication flow now includes the exact Vault secret path and explicit 401/403 error responses.
- The CORS policy now specifies exact allowed origins.
- List indentation now uses 4-space format as required.

### Issues Addressed from Critic Feedback
- **Edge Function Authentication Flow**: Added exact Vault secret path `secret/data/edge/api` and explicit HTTP 401/403 error JSON structures. Documented JWT extraction algorithm as HS256.
- **Markdown Syntax and Structure**: Changed list indentation to 4 spaces and ensured proper blank lines between paragraphs.
- **Security Hardening**: Added explicit allowed origins list for CORS policy and clarified expired token handling returns 401.

### Domain Insights
- Edge functions require tight integration with Vault for secure credential storage and retrieval.
- Proper error handling with standardized JSON responses is critical for API consumers to handle failures gracefully.
- Clear documentation of security policies (CORS, IP whitelisting) is essential for operational teams.

### Mermaid/C4 Syntax Rules Confirmed
- All container macros use Container()/ContainerDb() with proper parameters.
- System_Ext used correctly for external systems (Consul, Vault, Core llm-switch, Langfuse, Prometheus).
- Rel() macro used for all relationships with label and technology parameters.
- UpdateLayoutConfig must be last line.
- No -- or -> arrows allowed in C4 blocks.
- All string arguments use double quotes.
- Container labels with multiple lines use HTML <br> tags to comply with 'max 2 words per line' constraint.