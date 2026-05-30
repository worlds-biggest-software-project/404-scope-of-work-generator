# Data Model Suggestion 4: Knowledge Graph + Relational Hybrid (Neo4j + PostgreSQL)

> Project: Scope of Work Generator (404) · Generated: 2026-05-25

## Summary

A dual-database architecture combining PostgreSQL for transactional data (organisations, users, projects, rate cards, document metadata) with Neo4j as a knowledge graph for the clause library, document relationships, and AI-powered content recommendation. The knowledge graph captures the rich, interconnected relationships between clauses, SOW sections, client attributes, project types, risk factors, and historical outcomes that drive the SOW generator's organisational memory and intelligent content selection. This approach is purpose-built for the clause recommendation, risk detection, and scope-creep monitoring features that differentiate this product from generic AI writing tools.

---

## Key Entities and Relationships

### Architecture Overview

```
┌──────────────────────────────┐     ┌──────────────────────────────┐
│         PostgreSQL           │     │          Neo4j               │
│   (transactional backbone)   │     │   (knowledge graph)          │
│                              │     │                              │
│  organisation                │     │  (:Clause)──[:RELATED_TO]──▶(:Clause)
│  user                        │     │  (:Clause)──[:USED_IN]──▶(:SOWSection)
│  client                      │     │  (:Clause)──[:APPLIES_TO]──▶(:Vertical)
│  project                     │     │  (:Clause)──[:ADDRESSES]──▶(:RiskFactor)
│  rate_card / rate_card_entry │     │  (:SOWSection)──[:PART_OF]──▶(:SOWDocument)
│  sow_document (metadata)     │     │  (:SOWDocument)──[:FOR]──▶(:Client)
│  sow_version (metadata)      │     │  (:Client)──[:IN_INDUSTRY]──▶(:Industry)
│  approval_request            │     │  (:SOWDocument)──[:USES_TEMPLATE]──▶(:Template)
│  signature_request           │     │  (:RiskFactor)──[:MITIGATED_BY]──▶(:Clause)
│  crm_integration             │     │  (:Clause)──[:SUPERSEDES]──▶(:Clause)
│  audit_log                   │     │  (:SOWDocument)──[:LED_TO]──▶(:Outcome)
│                              │     │  (:Outcome)──[:HAD_ISSUE]──▶(:RiskFactor)
└──────────────────────────────┘     └──────────────────────────────┘
         │                                        │
         └──────── sync via graph_ref_id ─────────┘
```

### PostgreSQL Schema (Transactional Core)

```sql
-- ============================================================
-- ORGANISATION, USERS, CLIENTS, PROJECTS
-- (same normalised structure as Suggestion 1 — abbreviated)
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

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    contact_email   VARCHAR(320),
    crm_external_id VARCHAR(255),
    graph_node_id   VARCHAR(255),  -- reference to Neo4j node
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    vertical        VARCHAR(100),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Rate cards: same as Suggestion 1 (normalised tables for pricing integrity)
-- rate_card, rate_card_entry tables omitted for brevity — identical schema.

-- ============================================================
-- SOW DOCUMENT & VERSION METADATA
-- ============================================================

CREATE TABLE sow_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    document_number VARCHAR(50) UNIQUE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    current_version INTEGER NOT NULL DEFAULT 1,
    graph_node_id   VARCHAR(255),  -- reference to Neo4j node
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
    content         JSONB NOT NULL,  -- full document content (sections, line items, etc.)
    ai_context      JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version_number)
);

-- Approval, signature, CRM, and audit tables: same as Suggestions 1/3.
```

### Neo4j Knowledge Graph Schema

```cypher
// ============================================================
// NODE TYPES
// ============================================================

// --- Clause Nodes ---
// Each clause is a node with rich properties and relationships
CREATE CONSTRAINT clause_id IF NOT EXISTS FOR (c:Clause) REQUIRE c.id IS UNIQUE;

// Example clause node:
// (:Clause {
//     id: "uuid",
//     organisationId: "uuid",
//     category: "liability",
//     title: "Limitation of Liability",
//     body: "The total aggregate liability...",
//     version: 3,
//     requiresLegalApproval: true,
//     isActive: true,
//     createdAt: datetime(),
//     usageCount: 47,
//     successRate: 0.89  -- % of projects using this clause that closed successfully
// })

// --- SOW Document Nodes ---
CREATE CONSTRAINT sow_doc_id IF NOT EXISTS FOR (s:SOWDocument) REQUIRE s.id IS UNIQUE;

// (:SOWDocument {
//     id: "uuid",
//     organisationId: "uuid",
//     documentNumber: "SOW-2026-0042",
//     status: "signed",
//     totalValue: 150000.00,
//     vertical: "it_services",
//     createdAt: datetime()
// })

// --- SOW Section Nodes ---
CREATE CONSTRAINT sow_section_id IF NOT EXISTS FOR (ss:SOWSection) REQUIRE ss.id IS UNIQUE;

// (:SOWSection {
//     id: "uuid",
//     sectionType: "scope",
//     title: "Scope of Work",
//     bodyHash: "sha256...",  -- for deduplication and similarity detection
//     aiGenerated: true,
//     wordCount: 450
// })

// --- Client Nodes ---
CREATE CONSTRAINT client_id IF NOT EXISTS FOR (cl:Client) REQUIRE cl.id IS UNIQUE;

// (:Client {
//     id: "uuid",
//     name: "Acme Corp",
//     industry: "financial_services",
//     riskProfile: "medium",
//     totalProjectValue: 1250000.00,
//     projectCount: 8
// })

// --- Taxonomy Nodes ---
CREATE CONSTRAINT vertical_name IF NOT EXISTS FOR (v:Vertical) REQUIRE v.name IS UNIQUE;
CREATE CONSTRAINT risk_factor_id IF NOT EXISTS FOR (r:RiskFactor) REQUIRE r.id IS UNIQUE;
CREATE CONSTRAINT industry_name IF NOT EXISTS FOR (i:Industry) REQUIRE i.name IS UNIQUE;
CREATE CONSTRAINT outcome_id IF NOT EXISTS FOR (o:Outcome) REQUIRE o.id IS UNIQUE;

// (:Vertical { name: "it_services", description: "..." })
// (:Vertical { name: "construction", description: "..." })
// (:Industry { name: "financial_services", regulatoryRequirements: ["sox", "pci_dss"] })
// (:RiskFactor { id: "uuid", name: "ambiguous_scope", severity: "high", description: "..." })
// (:Outcome { id: "uuid", type: "scope_creep", impactValue: -25000.00, description: "..." })


// ============================================================
// RELATIONSHIP TYPES
// ============================================================

// Clause relationships
// (:Clause)-[:RELATED_TO { strength: 0.85 }]->(:Clause)
//   -- semantic similarity between clauses (computed by embedding model)

// (:Clause)-[:SUPERSEDES]->(:Clause)
//   -- version lineage: new clause replaces an older one

// (:Clause)-[:APPLIES_TO]->(:Vertical)
//   -- clause is relevant to a specific industry vertical

// (:Clause)-[:ADDRESSES]->(:RiskFactor)
//   -- clause mitigates a specific risk (e.g. liability clause addresses unlimited_liability risk)

// (:Clause)-[:USED_IN { sectionType: "payment_terms", addedAt: datetime() }]->(:SOWSection)
//   -- clause was used in a specific SOW section

// (:Clause)-[:RECOMMENDED_FOR]->(:Industry)
//   -- clause is commonly recommended for clients in this industry

// Document relationships
// (:SOWSection)-[:PART_OF { sortOrder: 1 }]->(:SOWDocument)
//   -- section belongs to a document

// (:SOWDocument)-[:FOR]->(:Client)
//   -- document was created for a client

// (:SOWDocument)-[:USES_TEMPLATE]->(:Template)

// (:SOWDocument)-[:IN_VERTICAL]->(:Vertical)

// (:SOWDocument)-[:PRECEDED_BY]->(:SOWDocument)
//   -- document is a revision of a previous SOW for the same engagement

// Outcome and learning relationships
// (:SOWDocument)-[:LED_TO]->(:Outcome)
//   -- signed SOW led to a specific project outcome (success, scope creep, dispute)

// (:Outcome)-[:HAD_ISSUE]->(:RiskFactor)
//   -- the outcome involved a specific risk factor

// (:Outcome)-[:COULD_HAVE_BEEN_PREVENTED_BY]->(:Clause)
//   -- retrospective: this clause could have prevented the negative outcome
```

### Key Graph Queries

```cypher
// ============================================================
// CLAUSE RECOMMENDATION: Given a new SOW for a client in
// financial services doing an IT project, recommend clauses
// that were used in successful past SOWs for similar contexts
// ============================================================

MATCH (c:Clause)-[:USED_IN]->(:SOWSection)-[:PART_OF]->(d:SOWDocument)
WHERE d.status = 'signed'
  AND d.organisationId = $orgId

// Filter by vertical and industry match
MATCH (d)-[:IN_VERTICAL]->(v:Vertical {name: $vertical})
MATCH (d)-[:FOR]->(cl:Client)-[:IN_INDUSTRY]->(i:Industry {name: $industry})

// Exclude clauses from SOWs that led to negative outcomes
OPTIONAL MATCH (d)-[:LED_TO]->(o:Outcome {type: 'scope_creep'})

WITH c, COUNT(DISTINCT d) AS usageCount, COUNT(o) AS negativeOutcomes
WHERE negativeOutcomes = 0

RETURN c.id, c.title, c.category, c.body, usageCount,
       c.successRate
ORDER BY usageCount DESC, c.successRate DESC
LIMIT 10;


// ============================================================
// RISK DETECTION: Given a draft SOW, find risk factors that
// are NOT addressed by any clause in the current document
// ============================================================

// Get all risk factors relevant to this vertical and industry
MATCH (rf:RiskFactor)<-[:ADDRESSES]-(c:Clause)-[:APPLIES_TO]->(v:Vertical {name: $vertical})

// Find which of these risk factors are NOT covered by clauses in the current SOW
WITH COLLECT(DISTINCT rf) AS allRisks, COLLECT(DISTINCT c) AS allMitigatingClauses

MATCH (currentSection:SOWSection)-[:PART_OF]->(currentDoc:SOWDocument {id: $documentId})
OPTIONAL MATCH (usedClause:Clause)-[:USED_IN]->(currentSection)

WITH allRisks, COLLECT(DISTINCT usedClause) AS usedClauses

UNWIND allRisks AS risk
MATCH (risk)<-[:ADDRESSES]-(mitigatingClause:Clause)
WHERE NOT mitigatingClause IN usedClauses

RETURN risk.name, risk.severity, risk.description,
       COLLECT(mitigatingClause.title) AS suggestedClauses
ORDER BY risk.severity DESC;


// ============================================================
// ORGANISATIONAL MEMORY: Find clauses similar to a given text
// using embedding-based similarity stored as relationship weight
// ============================================================

MATCH (c:Clause {organisationId: $orgId, isActive: true})
WHERE c.embedding IS NOT NULL

WITH c, gds.similarity.cosine(c.embedding, $inputEmbedding) AS similarity
WHERE similarity > 0.75

RETURN c.id, c.title, c.category, c.body, similarity
ORDER BY similarity DESC
LIMIT 5;


// ============================================================
// SCOPE CREEP PATTERN: Find which types of clauses are most
// commonly missing from SOWs that experienced scope creep
// ============================================================

MATCH (d:SOWDocument)-[:LED_TO]->(o:Outcome {type: 'scope_creep'})
WHERE d.organisationId = $orgId

MATCH (rf:RiskFactor)<-[:HAD_ISSUE]-(o)
MATCH (rf)<-[:ADDRESSES]-(c:Clause)

// Check if the clause was NOT used in the SOW
WHERE NOT EXISTS {
    MATCH (c)-[:USED_IN]->(:SOWSection)-[:PART_OF]->(d)
}

RETURN c.category, c.title, COUNT(DISTINCT d) AS missingFromCount,
       COLLECT(DISTINCT rf.name) AS relatedRisks
ORDER BY missingFromCount DESC
LIMIT 10;
```

### Synchronisation Between PostgreSQL and Neo4j

```typescript
// Sync service that keeps PostgreSQL and Neo4j consistent

interface GraphSyncService {
    // When a SOW document is created in PostgreSQL,
    // create the corresponding node and relationships in Neo4j
    onSOWDocumentCreated(doc: SOWDocument): Promise<void>;

    // When a clause is used in a SOW section,
    // create the USED_IN relationship in Neo4j
    onClauseUsedInSection(clauseId: string, sectionId: string, docId: string): Promise<void>;

    // When a project outcome is recorded (post-delivery),
    // create Outcome node and relationships for learning
    onProjectOutcomeRecorded(docId: string, outcome: ProjectOutcome): Promise<void>;

    // Nightly batch: recompute clause success rates, update embeddings,
    // recalculate relationship strengths
    runNightlyGraphEnrichment(orgId: string): Promise<void>;
}

// Implementation uses a transactional outbox pattern:
// 1. Write to PostgreSQL within a transaction
// 2. Write an outbox event to a pg_outbox table within the same transaction
// 3. A separate worker reads the outbox and applies changes to Neo4j
// 4. If Neo4j write fails, retry with backoff (eventually consistent)
```

---

## Pros

- **Superior clause recommendation.** Graph traversals find clauses that performed well in similar contexts (same vertical, same industry, same client risk profile) far more effectively than SQL queries across normalised join tables.
- **Natural risk factor modelling.** The relationships between risk factors, mitigating clauses, past outcomes, and verticals form a genuine graph. Neo4j traversals express this logic cleanly; SQL would require complex recursive CTEs.
- **Organisational memory as a first-class feature.** The knowledge graph captures what worked, what failed, and why — directly enabling the "learn from past SOWs" feature that differentiates this product from generic AI writers.
- **Embedding-native.** Neo4j's Graph Data Science library supports vector similarity search on clause embeddings, enabling semantic clause search and AI-powered content recommendation without a separate vector database.
- **Scope-creep pattern detection.** Graph pattern matching identifies which clause types are systematically missing from SOWs that experienced scope creep — a powerful analytical capability.
- **GraphRAG for AI generation.** When generating a SOW section with an LLM, the knowledge graph provides structured context (related clauses, risk factors, past outcomes) that dramatically improves generation quality compared to raw text retrieval.
- **Flexible taxonomy evolution.** Adding new verticals, risk factors, or outcome types is adding a node and relationships — no schema migration required.

## Cons

- **Dual-database operational complexity.** Two databases to operate, monitor, back up, and keep in sync. Significantly higher operational overhead than a PostgreSQL-only approach.
- **Consistency challenges.** PostgreSQL and Neo4j are separate systems. Keeping them consistent requires an outbox pattern, change data capture, or saga coordination — all adding complexity.
- **Team skill requirements.** Cypher (Neo4j's query language) is a specialised skill. Most developers know SQL; few know Cypher. Training and hiring costs are higher.
- **Neo4j licensing considerations.** Neo4j Community Edition is open source (GPLv3), but Enterprise Edition features (clustering, role-based access, advanced analytics) require a commercial licence. The GDS (Graph Data Science) library has a separate licence.
- **Graph enrichment latency.** Clause recommendation and risk detection queries depend on graph data being up-to-date. Nightly batch enrichment means recommendations may not reflect very recent SOW outcomes.
- **Not cost-effective for small deployments.** A consulting firm with a few hundred SOWs does not have enough graph data to make knowledge graph queries significantly better than simple SQL queries. The value grows with data volume.
- **ACID transactions span two systems.** Creating a SOW with rate card lookups (PostgreSQL) and clause recommendations (Neo4j) requires coordinating across databases. Latency and failure modes are more complex.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Transactional DB | PostgreSQL 16+ for all CRUD operations, rate cards, approval workflows |
| Knowledge Graph | Neo4j 5.x (Community for MVP, Enterprise for production clustering) |
| Vector Search | Neo4j GDS vector index for clause embeddings; or pgvector in PostgreSQL |
| Sync Layer | Debezium CDC from PostgreSQL to Neo4j, or transactional outbox + worker |
| Embedding Model | OpenAI text-embedding-3-large or Cohere embed-v3 for clause embeddings |
| API Layer | REST/GraphQL for CRUD; dedicated recommendation API backed by Neo4j |
| Search | Neo4j full-text indexes for clause search; PostgreSQL FTS for document search |
| File Storage | S3-compatible store for PDF/Word exports |
| Monitoring | Neo4j Ops Manager + PostgreSQL pg_stat_statements |

---

## Migration and Scaling Considerations

- **Start with PostgreSQL only, add Neo4j later.** Build the MVP with Suggestion 1 or 3. Once the product has enough clause usage data and project outcome history (6-12 months), introduce Neo4j for the recommendation and risk detection features. The graph is an enhancement layer, not a prerequisite.
- **Graph seeding.** When introducing Neo4j, seed the graph from existing PostgreSQL data: import clause records, SOW document metadata, and client/project relationships. Build USED_IN relationships from historical SOW section-to-clause mappings.
- **Embedding pipeline.** Run an asynchronous embedding pipeline that computes vector embeddings for every clause body and stores them on the Neo4j clause nodes. Recompute embeddings when clause versions change.
- **Read-through caching.** Cache frequently-accessed graph query results (clause recommendations for popular verticals) in Redis with a TTL. Most consulting firms operate in 2-3 verticals — the recommendation set is stable.
- **Neo4j clustering.** For production workloads, use Neo4j Aura (managed) or a self-hosted 3-node cluster for high availability. The knowledge graph is a read-heavy workload; use read replicas for recommendation queries.
- **Graph data residency.** For organisations with data residency requirements, the Neo4j graph can be deployed in the same region as the PostgreSQL database. Neo4j Aura supports region selection.
- **Fallback strategy.** If Neo4j is unavailable, the application should fall back to PostgreSQL-based clause queries (less sophisticated but functional). The knowledge graph is an enhancement, not a hard dependency.
- **Future: federated graphs.** If the product grows to serve multiple organisations that opt in to anonymised benchmarking, a federated graph can aggregate clause effectiveness data across organisations while preserving tenant isolation.
