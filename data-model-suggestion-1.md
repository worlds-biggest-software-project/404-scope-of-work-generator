# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Scope of Work Generator (404) · Generated: 2026-05-25

## Summary

A fully normalized relational data model using PostgreSQL, following Third Normal Form (3NF) conventions. Every entity — organisations, users, clients, projects, SOW documents, sections, clauses, rate cards, approvals, and signatures — is modelled as a dedicated table with foreign key relationships. This approach prioritises data integrity, referential consistency, and mature tooling over schema flexibility.

---

## Key Entities and Relationships

### Entity Relationship Overview

```
Organisation ──< User
Organisation ──< Client
Organisation ──< RateCard ──< RateCardEntry
Organisation ──< ClauseLibrary ──< Clause ──< ClauseVersion
Organisation ──< Template ──< TemplateSection

Client ──< Project
Project ──< SOWDocument
SOWDocument ──< SOWVersion
SOWVersion ──< SOWSection
SOWSection ──< SOWSectionComment
SOWSection >── Clause (optional reference)
SOWVersion ──< SOWLineItem >── RateCardEntry
SOWVersion ──< SOWAssumption
SOWVersion ──< SOWRiskFlag
SOWVersion ──< ApprovalRequest ──< ApprovalAction
SOWDocument ──< SignatureRequest ──< SignatureEvent

User ──< AuditLogEntry
CRMIntegration >── Organisation
```

### Core Schema

```sql
-- ============================================================
-- ORGANISATION & USERS
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(320) UNIQUE NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CLIENTS & PROJECTS
-- ============================================================

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    contact_email   VARCHAR(320),
    crm_external_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    vertical        VARCHAR(100),  -- IT services, consulting, creative, construction
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RATE CARDS
-- ============================================================

CREATE TABLE rate_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rate_card_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_card_id    UUID NOT NULL REFERENCES rate_card(id),
    role_title      VARCHAR(255) NOT NULL,
    hourly_rate     NUMERIC(12,2) NOT NULL,
    daily_rate      NUMERIC(12,2),
    billing_code    VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CLAUSE LIBRARY
-- ============================================================

CREATE TABLE clause_library (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE clause (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES clause_library(id),
    category        VARCHAR(100) NOT NULL, -- e.g. 'payment_terms', 'liability', 'ip_ownership'
    title           VARCHAR(255) NOT NULL,
    requires_legal_approval BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE clause_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clause_id       UUID NOT NULL REFERENCES clause(id),
    version_number  INTEGER NOT NULL,
    body            TEXT NOT NULL,
    approved_by     UUID REFERENCES "user"(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (clause_id, version_number)
);

-- ============================================================
-- TEMPLATES
-- ============================================================

CREATE TABLE template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    vertical        VARCHAR(100),
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE template_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES template(id),
    section_type    VARCHAR(100) NOT NULL, -- scope, deliverables, timeline, etc.
    title           VARCHAR(255) NOT NULL,
    default_content TEXT,
    sort_order      INTEGER NOT NULL,
    is_required     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SOW DOCUMENTS & VERSIONS
-- ============================================================

CREATE TABLE sow_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    template_id     UUID REFERENCES template(id),
    document_number VARCHAR(50) UNIQUE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- status: draft, internal_review, client_review, approved, signed, archived
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sow_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES sow_document(id),
    version_number  INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    executive_summary TEXT,
    total_value     NUMERIC(14,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version_number)
);

CREATE TABLE sow_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    section_type    VARCHAR(100) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL,
    source_clause_version_id UUID REFERENCES clause_version(id),
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sow_section_comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES sow_section(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    body            TEXT NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LINE ITEMS & PRICING
-- ============================================================

CREATE TABLE sow_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    description     VARCHAR(500) NOT NULL,
    rate_card_entry_id UUID REFERENCES rate_card_entry(id),
    quantity        NUMERIC(10,2) NOT NULL,
    unit            VARCHAR(50) NOT NULL DEFAULT 'hours',
    unit_rate       NUMERIC(12,2) NOT NULL,
    total           NUMERIC(14,2) GENERATED ALWAYS AS (quantity * unit_rate) STORED,
    billing_code    VARCHAR(50),
    sort_order      INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ASSUMPTIONS & RISK FLAGS
-- ============================================================

CREATE TABLE sow_assumption (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    body            TEXT NOT NULL,
    category        VARCHAR(100), -- technical, commercial, timeline, resource
    sort_order      INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sow_risk_flag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    section_id      UUID REFERENCES sow_section(id),
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium', -- low, medium, high
    flag_type       VARCHAR(100) NOT NULL, -- missing_assumption, ambiguous_scope, undefined_deliverable, inconsistency
    description     TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- APPROVAL & SIGNATURE WORKFLOWS
-- ============================================================

CREATE TABLE approval_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES approval_request(id),
    approver_id     UUID NOT NULL REFERENCES "user"(id),
    decision        VARCHAR(20) NOT NULL, -- approved, rejected, changes_requested
    comments        TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signature_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES sow_document(id),
    provider        VARCHAR(50) NOT NULL, -- docusign, adobe_sign, dropbox_sign
    external_envelope_id VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE signature_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES signature_request(id),
    signer_email    VARCHAR(320) NOT NULL,
    signer_name     VARCHAR(255),
    event_type      VARCHAR(50) NOT NULL, -- sent, viewed, signed, declined
    occurred_at     TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- CRM INTEGRATION
-- ============================================================

CREATE TABLE crm_integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        VARCHAR(50) NOT NULL, -- salesforce, hubspot, pipedrive
    external_account_id VARCHAR(255),
    access_token_encrypted BYTEA,
    refresh_token_encrypted BYTEA,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crm_trigger_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES crm_integration(id),
    trigger_stage   VARCHAR(255) NOT NULL, -- e.g. 'Proposal', 'Negotiation'
    template_id     UUID REFERENCES template(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AI GENERATION LOG
-- ============================================================

CREATE TABLE ai_generation_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES sow_version(id),
    section_id      UUID REFERENCES sow_section(id),
    model           VARCHAR(100) NOT NULL,  -- e.g. 'claude-opus-4-6'
    prompt_hash     VARCHAR(64),
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    generation_time_ms INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES "user"(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL, -- created, updated, deleted, approved, signed
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_org_time ON audit_log(organisation_id, created_at DESC);
```

---

## Pros

- **Referential integrity enforced at the database level.** Foreign keys, unique constraints, and check constraints prevent orphan records and data corruption. Critical for a system producing legally significant documents.
- **Mature, battle-tested tooling.** PostgreSQL has decades of production use, excellent ORM support (Prisma, Drizzle, TypeORM, SQLAlchemy), migration tooling (Flyway, Alembic, Prisma Migrate), and monitoring.
- **Straightforward querying.** JOINs across normalised tables are well-understood by developers. Reporting queries (e.g. "average time-to-SOW by client industry") are simple SQL.
- **ACID transactions.** Multi-table operations (creating a version with its sections, line items, and assumptions) are atomic. No partial document states.
- **Strong access control.** Row-level security (RLS) in PostgreSQL can enforce multi-tenant isolation at the database layer, preventing cross-organisation data leaks.
- **Compliance-friendly.** Normalised audit logs with foreign key references to entities satisfy ISO 9001 and GDPR audit trail requirements.

## Cons

- **Schema rigidity.** Adding a new SOW section type, a new metadata field, or industry-specific attributes requires a migration. This can slow iteration during early product development.
- **N+1 query risk.** Loading a complete SOW document (document + version + sections + line items + assumptions + risk flags + comments) requires many JOINs or multiple queries. Requires careful query planning or a DataLoader pattern.
- **Versioning complexity.** Full document versioning means duplicating all section and line item rows for each version. Storage grows linearly with version count. Diff computation must be done at the application layer.
- **Limited flexibility for AI outputs.** LLM-generated content may include structured metadata (confidence scores, alternative suggestions, reasoning traces) that does not fit neatly into fixed columns.
- **Horizontal scaling ceiling.** While PostgreSQL handles millions of documents well, sharding a highly normalised schema across multiple nodes is non-trivial compared to document-oriented stores.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Database | PostgreSQL 16+ with RLS enabled |
| ORM / Query Builder | Prisma (TypeScript) or SQLAlchemy (Python) |
| Migrations | Prisma Migrate or Alembic |
| Search | PostgreSQL full-text search (tsvector) for clause and document search; Typesense or Meilisearch for faceted search at scale |
| Caching | Redis for session state and hot rate card lookups |
| File storage | S3-compatible object store (AWS S3, MinIO) for PDF/Word exports |
| API layer | REST with OpenAPI 3.1 specification |

---

## Migration and Scaling Considerations

- **Initial deployment:** A single PostgreSQL instance handles the workload for small-to-mid consulting firms (thousands of SOWs, tens of thousands of versions). No sharding needed until significant scale.
- **Read replicas:** Add read replicas for reporting, search indexing, and AI training data extraction without impacting write performance.
- **Partitioning:** Partition `audit_log` and `ai_generation_log` tables by month using PostgreSQL native partitioning once they grow beyond tens of millions of rows.
- **Data archival:** Move signed, archived SOW documents and their versions to cold storage tables or an object store after a configurable retention period, retaining metadata in the primary tables.
- **Multi-region:** Use PostgreSQL logical replication to maintain read replicas in multiple regions for global consulting firms. Write operations remain centralised.
- **Migration from MVP:** This schema is designed to be the MVP starting point. If the product later moves to a hybrid or event-sourced model, the normalised schema serves as a clean source for data migration.
