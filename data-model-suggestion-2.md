# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Scope of Work Generator (404) · Generated: 2026-05-25

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every change to a SOW document, clause, rate card, or approval workflow is recorded as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Separate read-optimised projections (materialised views) serve queries for document display, search, and reporting. This approach provides a complete, auditable history of every change by design — particularly valuable for a system producing legally significant documents where the evolution of terms matters.

---

## Key Entities and Relationships

### Architecture Overview

```
                    ┌─────────────┐
                    │  Commands   │
                    │  (writes)   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Aggregates │
                    │  (domain)   │
                    └──────┬──────┘
                           │ emit events
                    ┌──────▼──────┐
                    │ Event Store │  ← append-only, immutable
                    │ (PostgreSQL │
                    │  or EventStoreDB)
                    └──────┬──────┘
                           │ project
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼────┐ ┌─────▼─────┐
       │ Document    │ │Search │ │ Analytics │
       │ Read Model  │ │ Index │ │ Warehouse │
       │ (PostgreSQL)│ │(Elastic)│ │ (ClickHouse)
       └─────────────┘ └───────┘ └───────────┘
```

### Aggregates (Write Side)

The domain is decomposed into aggregates that enforce business invariants:

| Aggregate | Responsibilities |
|-----------|-----------------|
| `SOWDocument` | Lifecycle of a single SOW: create, add/edit sections, update pricing, submit for review, approve, sign |
| `ClauseLibrary` | Manage clause definitions, versions, approval status |
| `RateCard` | Rate card entries, effective date ranges, activation |
| `Project` | Client-project association, vertical classification |
| `Organisation` | Tenant settings, user management |
| `ApprovalWorkflow` | Internal review routing, approval/rejection decisions |

### Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (append-only)
-- ============================================================

CREATE TABLE event_store (
    global_position   BIGSERIAL PRIMARY KEY,
    stream_id         VARCHAR(255) NOT NULL,  -- e.g. 'SOWDocument-<uuid>'
    stream_position   INTEGER NOT NULL,
    event_type        VARCHAR(255) NOT NULL,
    event_data        JSONB NOT NULL,
    metadata          JSONB NOT NULL DEFAULT '{}',
    -- metadata: { user_id, correlation_id, causation_id, ip_address, timestamp }
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, stream_position)
);

CREATE INDEX idx_event_store_type ON event_store(event_type);
CREATE INDEX idx_event_store_stream ON event_store(stream_id, stream_position);
CREATE INDEX idx_event_store_time ON event_store(created_at);

-- ============================================================
-- SNAPSHOT STORE (optional, for performance)
-- ============================================================

CREATE TABLE snapshot_store (
    stream_id         VARCHAR(255) PRIMARY KEY,
    stream_position   INTEGER NOT NULL,
    snapshot_data     JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain Events

```typescript
// === SOW Document Events ===

interface SOWDocumentCreated {
  type: 'SOWDocumentCreated';
  data: {
    documentId: string;
    projectId: string;
    templateId?: string;
    documentNumber: string;
    createdBy: string;
    brief: string;  // original client brief
  };
}

interface SOWSectionAdded {
  type: 'SOWSectionAdded';
  data: {
    documentId: string;
    sectionId: string;
    sectionType: string;  // scope, deliverables, timeline, etc.
    title: string;
    body: string;
    sortOrder: number;
    sourceClauseId?: string;
    aiGenerated: boolean;
  };
}

interface SOWSectionEdited {
  type: 'SOWSectionEdited';
  data: {
    documentId: string;
    sectionId: string;
    previousBody: string;
    newBody: string;
    editedBy: string;
    editReason?: string;
  };
}

interface SOWLineItemAdded {
  type: 'SOWLineItemAdded';
  data: {
    documentId: string;
    lineItemId: string;
    description: string;
    rateCardEntryId?: string;
    quantity: number;
    unit: string;
    unitRate: number;
    billingCode?: string;
  };
}

interface SOWRiskFlagged {
  type: 'SOWRiskFlagged';
  data: {
    documentId: string;
    flagId: string;
    sectionId?: string;
    severity: 'low' | 'medium' | 'high';
    flagType: string;
    description: string;
  };
}

interface SOWVersionPublished {
  type: 'SOWVersionPublished';
  data: {
    documentId: string;
    versionNumber: number;
    totalValue: number;
    publishedBy: string;
  };
}

interface SOWSubmittedForApproval {
  type: 'SOWSubmittedForApproval';
  data: {
    documentId: string;
    submittedBy: string;
    approverIds: string[];
  };
}

interface SOWApproved {
  type: 'SOWApproved';
  data: {
    documentId: string;
    approvedBy: string;
    comments?: string;
  };
}

interface SOWSentForSignature {
  type: 'SOWSentForSignature';
  data: {
    documentId: string;
    provider: string;
    externalEnvelopeId: string;
    signerEmails: string[];
  };
}

interface SOWSigned {
  type: 'SOWSigned';
  data: {
    documentId: string;
    signerEmail: string;
    signedAt: string;
  };
}

// === Clause Library Events ===

interface ClauseCreated {
  type: 'ClauseCreated';
  data: {
    clauseId: string;
    libraryId: string;
    category: string;
    title: string;
    body: string;
    requiresLegalApproval: boolean;
    createdBy: string;
  };
}

interface ClauseVersionAdded {
  type: 'ClauseVersionAdded';
  data: {
    clauseId: string;
    versionNumber: number;
    body: string;
    createdBy: string;
  };
}

interface ClauseVersionApproved {
  type: 'ClauseVersionApproved';
  data: {
    clauseId: string;
    versionNumber: number;
    approvedBy: string;
  };
}

// === Rate Card Events ===

interface RateCardCreated {
  type: 'RateCardCreated';
  data: {
    rateCardId: string;
    organisationId: string;
    name: string;
    currency: string;
    effectiveFrom: string;
  };
}

interface RateCardEntryAdded {
  type: 'RateCardEntryAdded';
  data: {
    rateCardId: string;
    entryId: string;
    roleTitle: string;
    hourlyRate: number;
    billingCode?: string;
  };
}

// === AI Generation Events ===

interface AIGenerationRequested {
  type: 'AIGenerationRequested';
  data: {
    documentId: string;
    model: string;
    promptHash: string;
    briefText: string;
  };
}

interface AIGenerationCompleted {
  type: 'AIGenerationCompleted';
  data: {
    documentId: string;
    model: string;
    inputTokens: number;
    outputTokens: number;
    generationTimeMs: number;
    sectionsGenerated: string[];
  };
}
```

### Read Model Projections

```sql
-- ============================================================
-- READ MODEL: Current SOW Document View
-- ============================================================

CREATE TABLE rm_sow_document (
    id                UUID PRIMARY KEY,
    project_id        UUID NOT NULL,
    organisation_id   UUID NOT NULL,
    client_name       VARCHAR(255),
    document_number   VARCHAR(50) UNIQUE NOT NULL,
    title             VARCHAR(500),
    status            VARCHAR(50) NOT NULL,
    current_version   INTEGER NOT NULL DEFAULT 1,
    total_value       NUMERIC(14,2),
    currency          CHAR(3) DEFAULT 'USD',
    section_count     INTEGER DEFAULT 0,
    risk_flag_count   INTEGER DEFAULT 0,
    unresolved_risks  INTEGER DEFAULT 0,
    created_by_name   VARCHAR(255),
    created_at        TIMESTAMPTZ NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- READ MODEL: SOW Section View (denormalised)
-- ============================================================

CREATE TABLE rm_sow_section (
    id                UUID PRIMARY KEY,
    document_id       UUID NOT NULL,
    section_type      VARCHAR(100) NOT NULL,
    title             VARCHAR(255) NOT NULL,
    body              TEXT NOT NULL,
    sort_order        INTEGER NOT NULL,
    ai_generated      BOOLEAN NOT NULL DEFAULT false,
    source_clause     VARCHAR(255),
    comment_count     INTEGER DEFAULT 0,
    last_edited_by    VARCHAR(255),
    last_edited_at    TIMESTAMPTZ
);

CREATE INDEX idx_rm_section_doc ON rm_sow_section(document_id, sort_order);

-- ============================================================
-- READ MODEL: SOW Timeline View (for dashboard)
-- ============================================================

CREATE TABLE rm_sow_timeline (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id       UUID NOT NULL,
    organisation_id   UUID NOT NULL,
    event_type        VARCHAR(100) NOT NULL,
    event_summary     TEXT NOT NULL,
    actor_name        VARCHAR(255),
    occurred_at       TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_timeline_doc ON rm_sow_timeline(document_id, occurred_at DESC);

-- ============================================================
-- READ MODEL: Analytics Projection
-- ============================================================

CREATE TABLE rm_sow_analytics (
    organisation_id   UUID NOT NULL,
    month             DATE NOT NULL,
    sows_created      INTEGER DEFAULT 0,
    sows_signed       INTEGER DEFAULT 0,
    avg_time_to_sign_hours NUMERIC(10,2),
    total_value_signed NUMERIC(16,2),
    ai_generation_count INTEGER DEFAULT 0,
    PRIMARY KEY (organisation_id, month)
);
```

---

## Pros

- **Complete audit trail by design.** Every change to every entity is an immutable event. No separate audit log needed — the event store IS the audit trail. Ideal for legally significant documents where the negotiation history matters.
- **Natural fit for document versioning.** SOW versions are simply snapshots at specific event positions. Version comparison is event replay between two positions. No need to duplicate section rows for each version.
- **Temporal queries are trivial.** "What did this SOW look like on March 15th?" is answered by replaying events up to that timestamp. Critical for scope-creep detection.
- **Read model flexibility.** Each consumer (document editor, search, analytics, scope-creep monitor) gets its own optimised projection. New read models can be built from the same event stream without schema migrations.
- **Decoupled scaling.** Write throughput (event appends) and read throughput (projection queries) scale independently. Read-heavy workloads (document viewing, search) can be scaled without affecting writes.
- **AI traceability.** AI generation requests and completions are first-class events, making it possible to audit which content was AI-generated, with which model, and what the original brief was.
- **Event-driven integrations.** CRM sync, e-signature triggers, and billing system updates are natural consumers of the event stream — subscribe to relevant events rather than polling for changes.

## Cons

- **Operational complexity.** Event sourcing requires understanding aggregates, projections, eventual consistency, and idempotent event handlers. The team needs experience or training.
- **Eventual consistency in read models.** After a command is processed, the read model may take milliseconds to seconds to update. Users might not see their change immediately unless read-your-own-writes is implemented.
- **Event schema evolution is hard.** Once events are persisted, their schema cannot be changed retroactively. Adding fields, renaming events, or splitting aggregates requires upcasting strategies and careful versioning.
- **Higher infrastructure cost.** Requires an event store, projection infrastructure, and potentially a message broker (Kafka, RabbitMQ, or NATS) for event distribution to projections.
- **Debugging complexity.** Troubleshooting requires understanding the event stream, aggregate state, and projection state — three moving parts instead of one database.
- **Overkill for small teams.** A consulting firm SOW tool may not generate enough document throughput to justify the architectural complexity of full event sourcing.
- **Full-text search requires a separate system.** The event store is not queryable for ad-hoc text search; a projection to Elasticsearch or similar is required.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Event Store | EventStoreDB (purpose-built) or PostgreSQL with append-only table |
| Message Broker | NATS JetStream or Apache Kafka for event distribution |
| Write-side framework | Axon Framework (Java/Kotlin), Eventuous (.NET), or custom (TypeScript/Python) |
| Read Model DB | PostgreSQL for document projections; ClickHouse for analytics |
| Search Projection | Elasticsearch or Typesense for clause and document search |
| Snapshot Store | PostgreSQL or Redis for aggregate snapshots |
| API Layer | GraphQL for flexible read queries; REST for commands |
| File Storage | S3-compatible store for PDF/Word exports |

---

## Migration and Scaling Considerations

- **Start with PostgreSQL as the event store.** A single PostgreSQL table with append-only writes handles tens of thousands of events per second. Migrate to EventStoreDB only if event volume demands it.
- **Build projections incrementally.** Start with a single document-view projection. Add search, analytics, and scope-creep monitor projections as features ship.
- **Snapshot strategy.** For SOW documents with hundreds of events (many edits across negotiation rounds), snapshot aggregate state every 50-100 events to keep replay fast.
- **Event versioning from day one.** Include a `schemaVersion` field in event metadata. Implement an upcaster pipeline that transforms old event formats to current ones during replay.
- **Migration from a relational MVP.** If the product starts with a normalised relational model (Suggestion 1), migration to event sourcing can be done by: (a) writing dual-write adapters that emit events alongside relational writes, (b) building projections from the new event stream, (c) cutting over reads to projections, (d) removing relational writes.
- **Multi-tenant isolation.** Prefix stream IDs with the organisation ID (e.g. `org-<uuid>/SOWDocument-<uuid>`) to enable per-tenant event streams. This supports future per-tenant scaling and data residency requirements.
- **GDPR right to erasure.** Event sourcing and GDPR deletion are in tension. Implement crypto-shredding: encrypt event data with per-tenant keys and destroy the key on deletion request, rendering events unreadable without deleting them from the stream.
