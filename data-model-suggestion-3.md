# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Scope of Work Generator (404) · Generated: 2026-05-25

## Summary

A pragmatic hybrid architecture using PostgreSQL as the sole database, combining normalised relational tables for stable, frequently-queried entities (organisations, users, clients, projects, rate cards) with JSONB columns for flexible, evolving, and semi-structured data (SOW document content, section bodies, AI generation metadata, clause metadata, template configurations). This approach captures the referential integrity benefits of a relational model while accommodating the schema flexibility that a document generation platform needs — especially for AI-generated content, variable SOW section structures, and industry-specific customisations.

---

## Key Entities and Relationships

### Design Philosophy

The guiding principle is: **use columns for what you query and filter on; use JSONB for what varies by context or evolves rapidly.**

| Data Category | Storage Approach | Rationale |
|--------------|-----------------|-----------|
| Organisation, User, Client, Project | Normalised tables | Stable schemas, used in JOINs, filtered frequently |
| Rate Cards and Entries | Normalised tables | Pricing integrity is critical; computed totals need reliable types |
| SOW Document metadata | Normalised columns | Status, document number, version number — queried constantly |
| SOW Section content | JSONB column per section | Section structure varies by template and industry vertical |
| Clause Library | Normalised + JSONB | Clause identity is relational; clause body and metadata are JSONB |
| AI generation context | JSONB column | Model outputs, prompts, confidence scores, alternatives vary per generation |
| Template definitions | JSONB column | Template structures vary by vertical and evolve frequently |
| Audit log details | JSONB column | Change payloads are heterogeneous by entity type |

### Core Schema

```sql
-- ============================================================
-- ORGANISATION, USERS, CLIENTS, PROJECTS
-- (normalised — stable schemas, used in JOINs and filters)
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { default_currency, default_vertical, branding, ai_preferences }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(320) UNIQUE NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences: { notification_settings, default_template, ui_theme }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    contact_email   VARCHAR(320),
    crm_external_id VARCHAR(255),
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes: { billing_address, payment_terms, preferred_currency,
    --               past_project_tags, risk_profile, custom_fields }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    vertical        VARCHAR(100),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { crm_opportunity_id, deal_value, probability,
    --             industry_codes, custom_fields }
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RATE CARDS (normalised — pricing integrity is critical)
-- ============================================================

CREATE TABLE rate_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { approved_by, approval_date, notes, region }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rate_card_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_card_id    UUID NOT NULL REFERENCES rate_card(id),
    role_title      VARCHAR(255) NOT NULL,
    hourly_rate     NUMERIC(12,2) NOT NULL,
    daily_rate      NUMERIC(12,2),
    billing_code    VARCHAR(50),
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes: { seniority_level, department, location_factor, overtime_multiplier }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CLAUSE LIBRARY (hybrid: relational identity + JSONB content)
-- ============================================================

CREATE TABLE clause (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    category        VARCHAR(100) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    requires_legal_approval BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    current_version INTEGER NOT NULL DEFAULT 1,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    -- tags for filtering: ['ip_ownership', 'saas', 'construction', 'gdpr']
    content         JSONB NOT NULL,
    -- content: {
    --   body: "...",
    --   variables: [{ name: "client_name", type: "string", required: true }],
    --   usage_guidance: "...",
    --   applicable_verticals: ["it_services", "consulting"],
    --   legal_notes: "..."
    -- }
    version_history JSONB NOT NULL DEFAULT '[]',
    -- version_history: [
    --   { version: 1, body: "...", created_by: "uuid", created_at: "...", approved_by: "uuid" },
    --   { version: 2, body: "...", ... }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clause_org_category ON clause(organisation_id, category);
CREATE INDEX idx_clause_tags ON clause USING GIN(tags);
CREATE INDEX idx_clause_content ON clause USING GIN(content jsonb_path_ops);

-- ============================================================
-- TEMPLATES (JSONB-heavy — structure varies by vertical)
-- ============================================================

CREATE TABLE template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    vertical        VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL,
    -- definition: {
    --   sections: [
    --     {
    --       sectionType: "scope",
    --       title: "Scope of Work",
    --       defaultContent: "...",
    --       required: true,
    --       sortOrder: 1,
    --       clauseSuggestions: ["clause-uuid-1", "clause-uuid-2"],
    --       aiPromptHint: "Focus on deliverables and exclusions"
    --     },
    --     { sectionType: "deliverables", ... },
    --     { sectionType: "timeline", ... },
    --     { sectionType: "payment_terms", ... },
    --     { sectionType: "assumptions", ... },
    --     { sectionType: "acceptance_criteria", ... }
    --   ],
    --   pricingLayout: "table" | "milestone" | "fixed",
    --   customSections: [...]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SOW DOCUMENTS (hybrid: relational metadata + JSONB content)
-- ============================================================

CREATE TABLE sow_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    template_id     UUID REFERENCES template(id),
    document_number VARCHAR(50) UNIQUE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    current_version INTEGER NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sow_doc_org_status ON sow_document(organisation_id, status);
CREATE INDEX idx_sow_doc_project ON sow_document(project_id);

CREATE TABLE sow_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES sow_document(id),
    version_number  INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    executive_summary TEXT,

    -- Pricing (normalised for aggregation queries)
    total_value     NUMERIC(14,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,

    -- Full document content as JSONB
    content         JSONB NOT NULL,
    -- content: {
    --   sections: [
    --     {
    --       id: "uuid",
    --       sectionType: "scope",
    --       title: "Scope of Work",
    --       body: "...",
    --       sortOrder: 1,
    --       sourceClauseId: "uuid" | null,
    --       aiGenerated: true,
    --       comments: [
    --         { id: "uuid", userId: "uuid", body: "...", resolved: false, createdAt: "..." }
    --       ]
    --     },
    --     ...
    --   ],
    --   lineItems: [
    --     {
    --       id: "uuid",
    --       description: "Senior Consultant",
    --       rateCardEntryId: "uuid",
    --       quantity: 120,
    --       unit: "hours",
    --       unitRate: 250.00,
    --       total: 30000.00,
    --       billingCode: "SC-001"
    --     },
    --     ...
    --   ],
    --   assumptions: [
    --     { id: "uuid", body: "...", category: "technical" },
    --     ...
    --   ],
    --   riskFlags: [
    --     {
    --       id: "uuid",
    --       sectionId: "uuid",
    --       severity: "high",
    --       flagType: "ambiguous_scope",
    --       description: "...",
    --       resolved: false
    --     },
    --     ...
    --   ]
    -- }

    -- AI generation metadata
    ai_context      JSONB NOT NULL DEFAULT '{}',
    -- ai_context: {
    --   originalBrief: "...",
    --   model: "claude-opus-4-6",
    --   generationLog: [
    --     { timestamp: "...", action: "initial_generation", inputTokens: 1200, outputTokens: 4500 },
    --     { timestamp: "...", action: "revision", prompt: "Make the timeline more aggressive", ... }
    --   ],
    --   confidenceScores: { scope: 0.92, deliverables: 0.87, ... }
    -- }

    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version_number)
);

-- GIN index for searching within SOW content
CREATE INDEX idx_sow_version_content ON sow_version USING GIN(content jsonb_path_ops);

-- ============================================================
-- APPROVAL & SIGNATURE WORKFLOWS (normalised — workflow state)
-- ============================================================

CREATE TABLE approval_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES sow_document(id),
    version_number  INTEGER NOT NULL,
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    actions         JSONB NOT NULL DEFAULT '[]',
    -- actions: [
    --   { approverId: "uuid", decision: "approved", comments: "...", decidedAt: "..." },
    --   ...
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signature_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES sow_document(id),
    provider        VARCHAR(50) NOT NULL,
    external_envelope_id VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    events          JSONB NOT NULL DEFAULT '[]',
    -- events: [
    --   { signerEmail: "...", eventType: "sent", occurredAt: "..." },
    --   { signerEmail: "...", eventType: "viewed", occurredAt: "..." },
    --   { signerEmail: "...", eventType: "signed", occurredAt: "..." }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CRM INTEGRATION (normalised + JSONB config)
-- ============================================================

CREATE TABLE crm_integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config: {
    --   externalAccountId: "...",
    --   triggerRules: [
    --     { triggerStage: "Proposal", templateId: "uuid", autoGenerate: true }
    --   ],
    --   fieldMappings: { dealName: "project.name", dealValue: "sow.totalValue" }
    -- }
    credentials_encrypted BYTEA,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG (relational metadata + JSONB changes)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES "user"(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    changes         JSONB,
    -- changes: { before: {...}, after: {...} }  or event-specific payload
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_org_time ON audit_log(organisation_id, created_at DESC);
```

### Querying Examples

```sql
-- Find all SOW documents with unresolved high-severity risk flags
SELECT
    d.document_number,
    d.status,
    v.title,
    risk_flag->>'description' AS risk_description
FROM sow_document d
JOIN sow_version v ON v.document_id = d.id
    AND v.version_number = d.current_version
CROSS JOIN LATERAL jsonb_array_elements(v.content->'riskFlags') AS risk_flag
WHERE d.organisation_id = :org_id
  AND risk_flag->>'severity' = 'high'
  AND (risk_flag->>'resolved')::boolean = false;

-- Search for clauses containing specific text, filtered by category
SELECT id, title, category, content->>'body' AS body
FROM clause
WHERE organisation_id = :org_id
  AND category = 'liability'
  AND content->>'body' ILIKE '%limitation of liability%';

-- Get all sections from the current version of a SOW
SELECT
    section->>'id' AS section_id,
    section->>'sectionType' AS section_type,
    section->>'title' AS title,
    section->>'body' AS body,
    (section->>'aiGenerated')::boolean AS ai_generated
FROM sow_version v
CROSS JOIN LATERAL jsonb_array_elements(v.content->'sections') AS section
WHERE v.document_id = :doc_id
  AND v.version_number = (
      SELECT current_version FROM sow_document WHERE id = :doc_id
  )
ORDER BY (section->>'sortOrder')::int;

-- Calculate total SOW value by client industry
SELECT
    c.industry,
    COUNT(DISTINCT d.id) AS sow_count,
    SUM(v.total_value) AS total_value
FROM sow_document d
JOIN project p ON d.project_id = p.id
JOIN client c ON p.client_id = c.id
JOIN sow_version v ON v.document_id = d.id
    AND v.version_number = d.current_version
WHERE d.organisation_id = :org_id
  AND d.status = 'signed'
GROUP BY c.industry;
```

---

## Pros

- **Single database, no additional infrastructure.** PostgreSQL handles both relational integrity and document storage. No need for a separate document database, reducing operational overhead.
- **Best-of-both-worlds flexibility.** Stable entities (orgs, users, rate cards) get full relational guarantees. Evolving structures (SOW content, template definitions, AI metadata) use JSONB and can change without migrations.
- **Rapid iteration on document structure.** Adding a new SOW section type, a new risk flag category, or an AI confidence score field requires no schema migration — just update the application code that reads/writes the JSONB.
- **Efficient document versioning.** Each SOW version stores its complete content as a single JSONB document. Loading a version is a single row fetch. Version comparison is a JSONB diff at the application layer.
- **GIN-indexed JSONB queries.** PostgreSQL GIN indexes on JSONB columns support fast containment queries (`@>`), path queries, and full-text search within document content.
- **Natural fit for AI-generated content.** LLM outputs with variable structures (confidence scores, alternative suggestions, reasoning traces) fit naturally into JSONB without schema changes.
- **Reduced N+1 query problem.** A SOW version with all its sections, line items, assumptions, and risk flags is a single row read, not a multi-table JOIN.

## Cons

- **No referential integrity within JSONB.** A `sourceClauseId` inside a JSONB section cannot be enforced as a foreign key at the database level. Orphan references are possible if clauses are deleted.
- **JSONB write amplification.** Updating a single section body requires rewriting the entire JSONB `content` column. For SOW documents with large content payloads, this is less efficient than updating a single row in a normalised sections table.
- **Complex JSONB queries.** While powerful, JSONB query syntax (`->`, `->>`, `@>`, `jsonb_array_elements`) is less intuitive than simple JOINs. Developers need to learn PostgreSQL JSONB idioms.
- **Schema validation shifts to application.** Without database-level constraints on JSONB structure, data validation must be enforced by the application (JSON Schema, Zod, Pydantic). Bugs can introduce malformed documents.
- **GIN index write cost.** GIN indexes on large JSONB columns are expensive to maintain. Write-heavy workloads (frequent edits to SOW content during negotiation) will feel the impact.
- **Reporting queries can be awkward.** Aggregating across JSONB fields (e.g. "average number of risk flags per SOW by industry") requires JSONB extraction functions, which are less performant than column-based aggregation.
- **Harder to enforce data governance.** GDPR field-level deletion within a JSONB blob requires surgical JSONB manipulation rather than a simple column NULL.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Database | PostgreSQL 16+ (single database for both relational and document storage) |
| ORM / Query Builder | Prisma with raw SQL for JSONB queries, or Drizzle ORM (better JSONB support) |
| Schema Validation | Zod (TypeScript) or Pydantic (Python) for JSONB structure validation before writes |
| JSON Schema | Publish a JSON Schema definition for the SOW content structure; validate on write |
| Migrations | Prisma Migrate for relational tables; application-level migration scripts for JSONB schema evolution |
| Search | PostgreSQL full-text search on JSONB content fields; Typesense for advanced clause search |
| Caching | Redis for hot document content and rate card lookups |
| File Storage | S3-compatible store for PDF/Word exports |
| API Layer | REST with OpenAPI 3.1; consider GraphQL for flexible document content queries |

---

## Migration and Scaling Considerations

- **Natural evolution from normalised MVP.** If starting with Suggestion 1 (fully normalised), migrating to a hybrid model is straightforward: consolidate section, line item, assumption, and risk flag tables into JSONB columns on the version table. The relational envelope (orgs, users, rate cards, document metadata) remains unchanged.
- **JSONB schema versioning.** Include a `schemaVersion` field in every JSONB column. When the application reads a document, check the schema version and apply upgrade transformations if needed. This avoids backfill migrations.
- **Partial indexes for common queries.** Create expression indexes on frequently-queried JSONB paths:
  ```sql
  CREATE INDEX idx_sow_status_value ON sow_version((content->'riskFlags')) 
      WHERE (content->'riskFlags') IS NOT NULL;
  ```
- **TOAST compression.** PostgreSQL automatically compresses JSONB values larger than ~2KB using TOAST. SOW content documents will typically be 10-50KB, well within TOAST efficiency range.
- **Partitioning.** Partition `audit_log` by month. Partition `sow_version` by `created_at` year if version tables grow beyond millions of rows.
- **Read replicas.** Use PostgreSQL streaming replication for read replicas serving document viewing and search workloads.
- **Future document store extraction.** If JSONB query performance becomes a bottleneck at extreme scale, the JSONB content can be extracted to a dedicated document store (MongoDB, Couchbase) while keeping the relational envelope in PostgreSQL. The hybrid design makes this extraction clean.
