# Scope of Work Generator — Phased Development Plan

> Project: 404-scope-of-work-generator · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, an additive directory structure, and 10 phased deliverables. Each task carries a **Design** section (real types, schemas, signatures) and a **Testing** section (named scenarios with inputs and expected outcomes).

---

## Product Synthesis

**What it does.** Transforms a free-text client brief (plus optional rate cards, clause libraries, and past-project context) into a structured, client-ready Scope of Work document — covering scope, deliverables, responsibilities, timeline, payment terms, assumptions, and acceptance criteria — in minutes. It adds organisational memory (learning from a firm's own historical SOWs), rate-card-driven pricing, AI risk/gap detection, an approval and e-signature workflow, and CRM-triggered generation.

**Who uses it.** Mid-market professional services firms, consultancies, IT services companies, creative agencies, and construction firms that issue SOWs regularly. Personas: the *Consultant/Engagement Lead* (drafts and iterates), the *Legal/Finance Approver* (gatekeeps clauses and pricing), the *Admin* (manages templates, rate cards, integrations), and the *Client signer* (reviews and signs).

**Key differentiators.** Accessible price vs. six-figure enterprise tools; ecosystem-agnostic (not locked to ClickUp/M365/Salesforce); open canonical SOW JSON Schema; AI-native risk detection and organisational memory rather than bolt-on AI; rate-card integration to stop margin leakage.

**Deployment model.** Self-hosted / cloud / hybrid. Single deployable backend (REST API + worker) + Postgres + Redis + object store + a web SPA. LLM via pluggable provider (Anthropic / OpenAI / Gemini). Also exposes an MCP server for AI-agent consumers.

**Standards that shape the design.** OpenAPI 3.1 (published spec), JSON Schema 2020-12 (canonical SOW data model + AI structured output validation), RFC 9457 Problem Details (error responses), RFC 6749 OAuth 2.0 + RFC 7519 JWT (auth and CRM/e-sign integrations), CloudEvents 1.0 (lifecycle webhooks), ISO 32000-2 PDF 2.0 + PAdES/eIDAS (signed exports), OWASP API Security Top 10 (security baseline), GDPR + EU AI Act transparency (AI-generated content disclosure), MCP (agent tools).

**Data model choice.** Primary: **Suggestion 3 — Hybrid Relational + JSONB on a single PostgreSQL instance**. Stable, frequently-queried entities (organisation, user, client, project, rate cards, clauses, signature/approval workflow, audit) are normalised tables; volatile AI-generated content (SOW section bodies, AI metadata, template config, risk flags) lives in JSONB. This balances referential integrity for legally significant data with the schema flexibility AI content demands, and keeps ops to a single database for self-hosters. Suggestion 1's normalised tables are adopted verbatim for the stable entities. Suggestion 4's Neo4j knowledge graph is deferred to Phase 9 (organisational memory) as an optional, additive recommendation engine — not required for MVP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript on Node.js 22 | API/integration-heavy product (CRM, e-sign, webhooks, SPA). Single language across backend + frontend reduces context-switching; first-class SDKs exist for Anthropic, OpenAI, DocuSign, HubSpot, Salesforce. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput, native JSON Schema validation per route (reuses our canonical SOW schemas), auto-generates the OpenAPI 3.1 document required by standards.md. |
| Validation / schemas | Zod + `zod-to-json-schema` | One source of truth for runtime validation and the published JSON Schema (2020-12) / OpenAPI components. LLM structured-output schemas reuse the same Zod definitions. |
| Database | PostgreSQL 16 (hybrid relational + JSONB) | Per data-model Suggestion 3. RLS for multi-tenant isolation; `tsvector` full-text search over clauses/SOWs; `jsonb` for AI content; mature tooling. |
| ORM / migrations | Drizzle ORM + drizzle-kit | Typed SQL with first-class JSONB column support and explicit, reviewable SQL migrations (important for a legally-significant audit trail). |
| Task queue | BullMQ on Redis | Async workloads: LLM generation, PDF rendering, CRM polling, webhook dispatch, e-sign callbacks. Retries + backoff + dead-letter built in. |
| Cache / sessions | Redis 7 | Session store, hot rate-card lookups, BullMQ backend, idempotency keys. |
| LLM access | Provider abstraction over Anthropic SDK (default `claude-opus-4`), OpenAI, Gemini | No IP encumbrance (per research). Pluggable so self-hosters choose a provider; prompt caching on Anthropic for repeated clause/template context. |
| Object storage | S3-compatible (AWS S3 / MinIO) | Generated PDF/DOCX exports and uploaded source documents. MinIO for self-host parity. |
| PDF generation | `@react-pdf/renderer` (layout) + `pdf-lib` (PAdES/LTV signature embedding) | ISO 32000-2 PDF 2.0 output; pdf-lib handles signature placeholders for eIDAS/PAdES. |
| DOCX generation | `docx` (npm) | Word export for clients not using the portal. |
| Frontend | React 19 + Vite + TanStack Query + Tailwind + shadcn/ui | Document-editor SPA with section-level editing, comments, diff view. TanStack Query for server-state; shadcn for fast, accessible UI. |
| Rich text / sections | TipTap (ProseMirror) | Section-level editing, tracked changes, comment anchors. |
| Auth | OAuth 2.0 / OIDC (RFC 6749) + JWT (RFC 7519) via `@fastify/jwt`; bcrypt for local accounts | Standards-aligned; CRM and e-sign integrations require OAuth client credentials/auth-code flows. |
| Integrations | DocuSign + Dropbox Sign (e-sign), HubSpot + Salesforce (CRM), Stripe + QuickBooks (billing) | Per README + standards.md. Each behind an adapter interface so providers are swappable. |
| Eventing | CloudEvents 1.0 envelopes over outbound webhooks | Notify billing/PM/CRM on SOW lifecycle transitions. |
| Errors | RFC 9457 Problem Details (`application/problem+json`) | Consistent machine-readable API errors per standards.md. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Exposes `generate_sow_draft`, `lookup_rate_card`, `retrieve_past_project`, `list_clause_templates`, `submit_for_approval`. |
| Testing | Vitest (unit/integration), Supertest (HTTP), Playwright (E2E), Testcontainers (real Postgres/Redis) | Full pyramid; Testcontainers gives real-DB integration without polluting dev DB. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Lint, format, strict type-check in CI. |
| Package manager / monorepo | pnpm workspaces + Turborepo | `apps/api`, `apps/worker`, `apps/web`, `apps/mcp`, shared `packages/*`. |
| Containerisation | Docker + docker-compose | One-command self-host: api, worker, web, postgres, redis, minio. |
| CI | GitHub Actions | Lint, type-check, test, build images, publish OpenAPI artifact. |

### Project Structure

```
404-scope-of-work-generator/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml           # api, worker, web, postgres, redis, minio
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── openapi.json                 # generated artifact (committed in CI)
├── packages/
│   ├── schema/                  # canonical SOW JSON Schema + Zod sources
│   │   └── src/
│   │       ├── sow.ts           # SowDocument, SowSection, LineItem, Assumption…
│   │       ├── clause.ts
│   │       ├── rate-card.ts
│   │       ├── events.ts        # CloudEvents envelope + lifecycle event types
│   │       └── index.ts
│   ├── db/                      # Drizzle schema, migrations, client, RLS helpers
│   │   ├── src/
│   │   │   ├── schema.ts
│   │   │   ├── client.ts
│   │   │   └── rls.ts
│   │   └── drizzle/             # generated SQL migrations
│   ├── llm/                     # provider abstraction + prompt templates
│   │   └── src/
│   │       ├── provider.ts      # LlmProvider interface
│   │       ├── anthropic.ts
│   │       ├── openai.ts
│   │       ├── gemini.ts
│   │       └── prompts/         # generation, risk-detection, revision, scope-creep
│   ├── core/                    # domain services (no HTTP/queue concerns)
│   │   └── src/
│   │       ├── sow-service.ts
│   │       ├── generation-service.ts
│   │       ├── risk-service.ts
│   │       ├── pricing-service.ts
│   │       ├── clause-service.ts
│   │       ├── approval-service.ts
│   │       ├── export-service.ts
│   │       └── audit-service.ts
│   ├── integrations/            # adapters behind shared interfaces
│   │   └── src/
│   │       ├── esign/ (docusign.ts, dropbox-sign.ts, esign-port.ts)
│   │       ├── crm/   (hubspot.ts, salesforce.ts, crm-port.ts)
│   │       └── billing/ (stripe.ts, quickbooks.ts, billing-port.ts)
│   └── config/                  # env parsing (Zod), logger, problem-details helper
├── apps/
│   ├── api/                     # Fastify HTTP server
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/ (auth, rls-context, error-handler, swagger)
│   │       └── routes/ (auth, sow, sections, clauses, rate-cards, templates,
│   │                    approvals, signatures, crm, webhooks, export, search)
│   ├── worker/                  # BullMQ processors
│   │   └── src/
│   │       ├── index.ts
│   │       └── jobs/ (generate-sow, detect-risk, render-pdf, dispatch-webhook,
│   │                  poll-crm, esign-callback, scope-creep-scan)
│   ├── web/                     # React SPA
│   │   └── src/
│   │       ├── routes/
│   │       ├── components/
│   │       ├── editor/          # TipTap section editor, comments, diff
│   │       └── api/             # generated client from openapi.json
│   └── mcp/                     # MCP server exposing SOW tools
│       └── src/index.ts
├── tests/
│   ├── fixtures/                # sample briefs, rate cards, clause libs, golden SOWs
│   ├── integration/
│   └── e2e/
└── README.md
```

---

## Phase 1: Foundation — Monorepo, Schema Package, Database, Auth

### Purpose
Establish the workspace, the canonical SOW data model (the project's intended de-facto standard), the hybrid Postgres schema with RLS multi-tenancy, and authentication. After this phase the system can run, migrate, authenticate users, and validate SOW documents against published schemas — the substrate every later phase builds on.

### Tasks

#### 1.1 — Monorepo & tooling scaffold
**What**: pnpm + Turborepo workspace with shared TS config, ESLint, Prettier, Vitest, and a docker-compose for local infra.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- Root `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"module": "NodeNext"`, `"target": "ES2023"`.
- `turbo.json` pipelines: `build` (depends on `^build`), `lint`, `typecheck`, `test`.
- `docker-compose.yml` services: `postgres:16` (POSTGRES_DB=sow), `redis:7`, `minio` (with `createbuckets` init), exposing 5432/6379/9000.
- `packages/config/src/env.ts` parses env with Zod:
```ts
export const Env = z.object({
  NODE_ENV: z.enum(['development','test','production']).default('development'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  S3_ENDPOINT: z.string().url(), S3_BUCKET: z.string(),
  S3_ACCESS_KEY: z.string(), S3_SECRET_KEY: z.string(),
  LLM_PROVIDER: z.enum(['anthropic','openai','gemini']).default('anthropic'),
  LLM_API_KEY: z.string(),
  LLM_MODEL: z.string().default('claude-opus-4'),
});
export type Env = z.infer<typeof Env>;
```

**Testing**:
- `Unit: env.ts with all required vars set → parsed Env object with defaults applied (LLM_MODEL='claude-opus-4')`.
- `Unit: env.ts missing JWT_SECRET → throws ZodError naming 'JWT_SECRET'`.
- `Unit: env.ts JWT_SECRET length 10 → ZodError 'min 32'`.
- `Smoke: docker-compose up → postgres, redis, minio all report healthy within 60s`.

#### 1.2 — Canonical SOW schema package
**What**: Zod definitions for the SOW domain that emit JSON Schema 2020-12 and OpenAPI components.

**Design**:
```ts
export const SectionType = z.enum([
  'scope','deliverables','responsibilities','timeline',
  'payment_terms','assumptions','acceptance_criteria','out_of_scope'
]);

export const SowLineItem = z.object({
  id: z.string().uuid(),
  description: z.string().max(500),
  rateCardEntryId: z.string().uuid().nullable(),
  quantity: z.number().nonnegative(),
  unit: z.enum(['hours','days','fixed','units']).default('hours'),
  unitRate: z.number().nonnegative(),
  total: z.number().nonnegative(),       // quantity * unitRate, validated
  billingCode: z.string().nullable(),
});

export const SowSection = z.object({
  id: z.string().uuid(),
  type: SectionType,
  title: z.string().max(255),
  body: z.string(),                       // markdown
  sortOrder: z.number().int(),
  sourceClauseVersionId: z.string().uuid().nullable(),
  aiGenerated: z.boolean().default(false),
});

export const SowRiskFlag = z.object({
  id: z.string().uuid(),
  sectionId: z.string().uuid().nullable(),
  severity: z.enum(['low','medium','high']),
  flagType: z.enum(['missing_assumption','ambiguous_scope','undefined_deliverable','inconsistency','missing_acceptance_criteria']),
  description: z.string(),
  suggestedResolution: z.string().nullable(),
  isResolved: z.boolean().default(false),
});

export const SowContent = z.object({   // the JSONB payload of a version
  title: z.string().max(500),
  executiveSummary: z.string().nullable(),
  currency: z.string().length(3).default('USD'),
  startDate: z.string().date().nullable(),
  endDate: z.string().date().nullable(),
  sections: z.array(SowSection),
  lineItems: z.array(SowLineItem),
  assumptions: z.array(z.object({ id: z.string().uuid(), body: z.string(), category: z.string().nullable(), sortOrder: z.number().int() })),
  riskFlags: z.array(SowRiskFlag),
});
```
- `pnpm --filter @sow/schema build:json-schema` writes `schema/dist/sow.schema.json` (draft 2020-12) via `zod-to-json-schema`.

**Testing**:
- `Unit: SowContent.parse(valid fixture) → success`.
- `Unit: SowLineItem with total ≠ quantity*unitRate → custom refinement error`.
- `Unit: SowSection.type='invalid' → ZodError enumerating allowed types`.
- `Unit: generated sow.schema.json validates the same fixture under ajv 2020-12 dialect → no errors` (cross-checks Zod↔JSON Schema parity).

#### 1.3 — Database schema, migrations, RLS
**What**: Drizzle schema implementing the hybrid model; initial migration; RLS policies for tenant isolation.

**Design**: Normalised tables (from Suggestion 1, adopted): `organisation`, `user`, `client`, `project`, `rate_card`, `rate_card_entry`, `clause_library`, `clause`, `clause_version`, `template`, `sow_document`, `approval_request`, `approval_action`, `signature_request`, `signature_event`, `crm_integration`, `crm_trigger_rule`, `audit_log`. Hybrid/JSONB tables:
```sql
CREATE TABLE sow_version (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES sow_document(id),
  organisation_id UUID NOT NULL REFERENCES organisation(id),  -- for RLS
  version_number INTEGER NOT NULL,
  content JSONB NOT NULL,            -- validated SowContent payload
  total_value NUMERIC(14,2),
  ai_metadata JSONB DEFAULT '{}',    -- model, token counts, prompt hashes
  created_by UUID NOT NULL REFERENCES "user"(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (document_id, version_number)
);
CREATE TABLE template (
  ..., config JSONB NOT NULL DEFAULT '{}'  -- section scaffold, vertical terminology
);
CREATE INDEX idx_sow_version_content ON sow_version USING gin (content);
```
- Every tenant-scoped table carries `organisation_id`. RLS: `CREATE POLICY tenant_isolation ON <t> USING (organisation_id = current_setting('app.org_id')::uuid);` Enabled with `ALTER TABLE … ENABLE ROW LEVEL SECURITY`.
- `packages/db/src/rls.ts` exports `withOrg(orgId, fn)` that runs `SET LOCAL app.org_id` inside a transaction.

**Testing**:
- `Integration (Testcontainers Postgres): run all migrations → schema applies, expected tables exist`.
- `Integration: insert sow_version for org A; query as app.org_id=B → 0 rows (RLS isolation)`.
- `Integration: insert version with content failing SowContent shape is allowed at DB layer but rejected by service layer` (documents where validation lives).
- `Integration: duplicate (document_id, version_number) → unique violation`.

#### 1.4 — Auth & tenant context
**What**: Local email/password + JWT issuance, plus an OAuth2/OIDC-ready abstraction; Fastify plugin injecting org context for RLS.

**Design**:
- `POST /auth/register` (first user creates org+admin), `POST /auth/login` → `{ accessToken, refreshToken }` (JWT, RFC 7519; claims `sub`, `org`, `role`, `exp`).
- `@fastify/jwt` verifies; `rls-context` plugin reads `org` claim and wraps the request DB calls in `withOrg`.
- Roles: `admin | member | approver | viewer`. Authorisation guard `requireRole(...roles)`.
- Passwords bcrypt (cost 12). Refresh tokens stored hashed in Redis with TTL.

**Testing**:
- `Integration: register → 201, org + admin user created, JWT returned`.
- `Integration: login wrong password → RFC 9457 problem+json, status 401, type ".../invalid-credentials"`.
- `Integration: request to tenant route with org=A token → only org A rows visible`.
- `Unit: requireRole('approver') with member token → 403 problem+json`.

---

## Phase 2: SOW CRUD, Versioning & Document Service

### Purpose
Build the document lifecycle core: create SOW documents, store immutable versions, edit sections, and compute diffs between versions. This is the spine onto which AI generation, pricing, approvals, and signing attach. After this phase a user can manually author and version a SOW end-to-end with no AI.

### Tasks

#### 2.1 — SOW document & version CRUD
**What**: REST endpoints and `sow-service` for documents/versions.

**Design**:
- `POST /sows` `{ projectId, templateId? }` → creates `sow_document` (status `draft`, generates `document_number` `SOW-{YYYY}-{seq}`) + an initial empty `sow_version` v1 from template config.
- `GET /sows/:id` → document + latest version content.
- `GET /sows/:id/versions`, `GET /sows/:id/versions/:n`.
- `POST /sows/:id/versions` → snapshots current content into a new immutable version (version_number = max+1). Versions are never mutated after a newer one exists; edits target the latest draft version.
- `PATCH /sows/:id/versions/latest` `{ content: SowContent }` → validates with Zod, persists JSONB (only allowed while document.status='draft').
- Status state machine: `draft → internal_review → client_review → approved → signed → archived` (also `→ rejected` from review states back to `draft`). Transitions enforced in `sow-service.transition()`.

**Testing**:
- `Integration: POST /sows → 201, document_number matches /SOW-\d{4}-\d+/, v1 exists`.
- `Integration: PATCH latest with invalid content → 422 problem+json listing the failing field`.
- `Integration: PATCH latest when status='signed' → 409 (immutable)`.
- `Unit: transition('draft','signed') → InvalidTransition error; transition('draft','internal_review') → ok`.
- `Integration: POST new version after edits → version_number increments, prior version content unchanged`.

#### 2.2 — Section editing & comments
**What**: Section-level edit operations and threaded comments.

**Design**:
- Sections live inside `content.sections`. `PUT /sows/:id/sections/:sectionId` updates one section (title/body/sortOrder) in the latest draft version atomically.
- Comments are relational (queried/filtered often): `sow_section_comment(id, document_id, section_id, user_id, body, resolved, created_at)`.
- `POST /sows/:id/sections/:sectionId/comments`, `PATCH …/comments/:cid {resolved}`.

**Testing**:
- `Integration: PUT section reorders → content.sections sortOrder updated, others unchanged`.
- `Integration: PUT unknown sectionId → 404 problem+json`.
- `Integration: add comment then resolve → resolved=true, audit entry written`.

#### 2.3 — Version diffing
**What**: Structured diff between two versions for the negotiation comparison view.

**Design**:
```ts
type SectionDiff = { type: SectionType; status: 'added'|'removed'|'modified'|'unchanged';
  textDiff?: { added: string[]; removed: string[] } };
type VersionDiff = { sections: SectionDiff[]; lineItemsDelta: { addedTotal: number; removedTotal: number; netValueChange: number }; };
function diffVersions(a: SowContent, b: SowContent): VersionDiff
```
- Text diff via `diff` (Myers) at paragraph granularity; sections matched by `type`+`title`.
- `GET /sows/:id/diff?from=2&to=4` → `VersionDiff`.

**Testing**:
- `Unit: diff identical content → all sections 'unchanged', netValueChange 0`.
- `Unit: section body changed → status 'modified' with non-empty added/removed`.
- `Unit: line item added → lineItemsDelta.netValueChange equals its total`.
- `Integration: GET /diff?from=1&to=2 → 200 VersionDiff`.

---

## Phase 3: AI Generation Engine (Core Value Proposition)

### Purpose
The heart of the product: turn a free-text brief into a fully structured SOW using an LLM, with provider abstraction, structured-output validation against the canonical schema, and an async job so long generations don't block requests. After this phase the headline "brief → SOW in minutes" works.

### Tasks

#### 3.1 — LLM provider abstraction
**What**: A `LlmProvider` interface with Anthropic (default), OpenAI, and Gemini implementations supporting structured JSON output.

**Design**:
```ts
interface LlmProvider {
  generateStructured<T>(args: {
    system: string; user: string; schema: ZodType<T>;
    maxTokens?: number; cacheKey?: string;  // enables prompt caching
  }): Promise<{ data: T; usage: { inputTokens: number; outputTokens: number }; model: string }>;
}
```
- Anthropic impl uses tool-use/JSON mode; passes `schema` as the tool input schema (zod→JSON Schema), retries once on schema-invalid output, and uses prompt caching for the (large, stable) clause/template context block keyed by `cacheKey`.
- Selected via `LLM_PROVIDER`. EU AI Act transparency: every generated section is tagged `aiGenerated:true` and `ai_metadata` records model + token counts.

**Testing**:
- `Unit (mocked SDK): generateStructured returns malformed JSON once then valid → succeeds on retry, usage summed`.
- `Unit (mocked): returns content failing schema twice → throws LlmStructuredOutputError`.
- `Unit: provider selected by LLM_PROVIDER env`.

#### 3.2 — Generation prompts & service
**What**: `generation-service.generateSow(brief, context)` producing a validated `SowContent`.

**Design**:
- Context assembled from: template config (section scaffold + vertical terminology), retrieved clause snippets (Phase 5), and rate card roles (Phase 4) — all optional in MVP.
- System prompt template (`packages/llm/src/prompts/generate.ts`):
  > "You are an expert professional-services contracts author. Produce a Scope of Work as STRICT JSON matching the provided schema. Cover every required section: scope, deliverables, responsibilities, timeline, payment_terms, assumptions, acceptance_criteria. Be specific and measurable; never invent prices — leave line items empty unless rate data is provided. Mark each section you author. {vertical_guidance} {clause_guidance}"
- User prompt carries the raw brief + structured hints (`{ vertical, knownDeliverables?, durationHint? }`).
- Output validated against `SowContent`; UUIDs assigned server-side; `aiGenerated=true` on every produced section.

**Testing**:
- `Integration (mocked LLM returning golden JSON): generateSow(sample brief) → SowContent with all 7 required section types present`.
- `Unit: post-process assigns fresh UUIDs and sets aiGenerated=true on each section`.
- `Unit: line items empty when no rate context supplied (no invented prices)`.
- `Fixture: golden brief → snapshot of section titles stable`.

#### 3.3 — Async generation job & endpoint
**What**: `POST /sows/:id/generate` enqueues a BullMQ job; status pollable; result written as a new version.

**Design**:
- Request `{ brief: string, hints?: {...} }` → 202 Accepted + `{ jobId }` and `Location: /jobs/:jobId`.
- Worker `generate-sow` job: calls `generateSow`, writes new `sow_version`, records `ai_metadata`, emits `sow.version.generated` CloudEvent (dispatched in Phase 7).
- `GET /jobs/:jobId` → `{ status: 'queued'|'active'|'completed'|'failed', resultVersion? }`.
- Idempotency: optional `Idempotency-Key` header deduped in Redis (24h).

**Testing**:
- `Integration (real Redis + mocked LLM): POST /generate → 202 with jobId; worker processes; GET /jobs/:id eventually 'completed' with resultVersion`.
- `Integration: LLM error in job → job 'failed' with problem-detail payload; no partial version written`.
- `Integration: same Idempotency-Key twice → single job, same jobId returned`.

#### 3.4 — Conversational revision
**What**: `POST /sows/:id/revise` applies a natural-language instruction to the current draft.

**Design**:
- Request `{ instruction, sectionType? }`. Service sends current `SowContent` (or one section) + instruction to the LLM with a revision prompt that returns the modified section(s) only, merged into a new version.
- Prompt instructs the model to preserve untouched sections verbatim and return a JSON patch list `[{ sectionType, newBody }]`.

**Testing**:
- `Integration (mocked LLM): revise "make timeline 8 weeks" → only timeline section body changed, others identical, new version created`.
- `Unit: patch targeting non-existent sectionType → ignored with warning, no crash`.

---

## Phase 4: Rate Cards & Pricing

### Purpose
Eliminate manual rate entry and margin leakage by sourcing pricing from approved rate cards and auto-computing line-item and document totals. After this phase generated SOWs can carry accurate, sourced pricing.

### Tasks

#### 4.1 — Rate card management
**What**: CRUD for rate cards and entries with effective-dating.

**Design**:
- `rate_card(id, organisation_id, name, currency, effective_from, effective_to, is_active)`; `rate_card_entry(id, rate_card_id, role_title, hourly_rate, daily_rate, billing_code)`.
- `POST/GET/PATCH /rate-cards`, `…/:id/entries`. Only one active card per currency by default (overlapping effective ranges warned).
- `GET /rate-cards/active?currency=USD&on=2026-06-01` → the entry set effective on that date.

**Testing**:
- `Integration: create card + entries → retrievable; entries carry billing codes`.
- `Unit: overlapping effective ranges for same currency → validation warning surfaced`.
- `Integration: active lookup picks the card whose range contains the date`.

#### 4.2 — Pricing service & line-item population
**What**: Compute totals and resolve line items against rate-card entries.

**Design**:
```ts
function priceLineItems(items: SowLineItem[], card: RateCardEntry[]):
  { items: SowLineItem[]; documentTotal: number; warnings: string[] }
```
- For each line item with `rateCardEntryId`, sets `unitRate` from the card (rejects manual override mismatch unless `allowOverride`), recomputes `total`, and `documentTotal = Σ total`. Items referencing missing/expired entries produce a warning and are flagged.
- Writes `sow_version.total_value`.

**Testing**:
- `Unit: two items 10h@150 + 5h@200 → totals 1500/1000, documentTotal 2500`.
- `Unit: item references inactive entry → warning, item flagged, excluded from confident total`.
- `Unit: manual unitRate ≠ card rate without allowOverride → mismatch warning`.

#### 4.3 — Pricing-aware generation
**What**: Feed rate roles into generation so the LLM proposes quantities against real roles.

**Design**:
- Generation context includes available roles (`role_title`, `unit`, `billingCode`) but NOT a mandate to invent totals; the LLM proposes `{ description, roleTitle, quantity, unit }`, then `pricing-service` resolves rates server-side (model never sets prices).

**Testing**:
- `Integration (mocked LLM): generate with rate card → line items have server-resolved unitRate matching the card, model-provided prices ignored`.
- `Unit: model proposes unknown roleTitle → item kept with null rateCardEntryId + warning`.

---

## Phase 5: Clause Library & AI Risk/Gap Detection

### Purpose
Add legal guardrails (versioned, approval-flagged clause library + compliance check) and the AI risk detector that flags missing assumptions, ambiguous scope, and undefined deliverables before a document is sent. These are the trust-and-quality differentiators.

### Tasks

#### 5.1 — Clause library with versioning & approval flags
**What**: CRUD for clause libraries, clauses, and immutable clause versions; full-text search.

**Design**:
- Tables per Suggestion 1: `clause_library`, `clause(category, title, requires_legal_approval, is_active)`, `clause_version(version_number, body, approved_by, approved_at, created_by)`.
- `tsvector` index over `clause_version.body` + `clause.title`. `GET /clauses/search?q=&category=&approvedOnly=true`.
- Inserting a SOW section from a clause sets `sourceClauseVersionId`.

**Testing**:
- `Integration: create clause v1, edit → v2 created, v1 immutable`.
- `Integration: search "indemnity" → ranked matches; approvedOnly excludes unapproved versions`.
- `Integration: approve clause version sets approved_by/approved_at + audit entry`.

#### 5.2 — Compliance checker
**What**: Flag SOW sections whose text deviates from approved clause language.

**Design**:
```ts
function checkCompliance(content: SowContent, approved: ClauseVersion[]):
  { deviations: { sectionId: string; clauseVersionId: string; similarity: number; note: string }[] }
```
- For sections referencing a clause, compute similarity (token-set / cosine over embeddings if available, else trigram) between section body and the approved clause body; below threshold (default 0.85) → deviation. Also flags use of any clause whose latest version is unapproved.
- `GET /sows/:id/compliance` → deviations list.

**Testing**:
- `Unit: section matching approved clause verbatim → no deviation`.
- `Unit: section materially altered from approved clause → deviation with similarity < 0.85`.
- `Unit: references clause version pending approval → deviation note 'unapproved language'`.

#### 5.3 — AI risk & gap detection
**What**: `risk-service.detectRisks(content)` returns `SowRiskFlag[]`, persisted on the version.

**Design**:
- LLM call with a risk-detection prompt + the `SowRiskFlag[]` schema as structured output:
  > "Review this SOW. Identify: missing or undefined deliverables, ambiguous scope language, missing assumptions, internal inconsistencies (e.g. timeline vs. payment schedule), and missing acceptance criteria. Return a JSON array of flags with severity and a concrete suggested resolution. Do not invent issues that are not supported by the text."
- Deterministic pre-checks (no LLM) also run: required section present? acceptance_criteria empty? timeline references dates outside start/end? These guarantee baseline flags even if the LLM under-reports.
- `POST /sows/:id/detect-risks` (async, like generation) → flags written into `content.riskFlags`; `PATCH …/risk-flags/:id {isResolved}`.

**Testing**:
- `Unit (deterministic): SOW missing acceptance_criteria section → at least one 'missing_acceptance_criteria' high flag`.
- `Integration (mocked LLM): brief with vague scope → flags include 'ambiguous_scope'`.
- `Unit: end_date before start_date → 'inconsistency' flag from deterministic check`.
- `Integration: resolve a flag → isResolved=true persisted`.

---

## Phase 6: Approval Workflow, Export & E-Signature

### Purpose
Take a finished draft through internal sign-off, render it to standards-compliant PDF/DOCX, and execute it via e-signature. After this phase a SOW can travel the full path from approved internal draft to a legally signed document.

### Tasks

#### 6.1 — Approval routing
**What**: Configurable internal approval requests and decisions.

**Design**:
- `approval_request(version_id, requested_by, status)`, `approval_action(request_id, approver_id, decision, comments)`. Decision ∈ `approved|rejected|changes_requested`.
- `POST /sows/:id/submit-for-approval` (transitions document → `internal_review`, creates request, notifies approvers). `POST /approvals/:id/decide`. All required approvers must approve → document → `approved`; any rejection → `draft`.
- Approval rule config on template/org: list of required approver roles.

**Testing**:
- `Integration: submit → status internal_review, approver notified (job enqueued)`.
- `Integration: single required approver approves → document 'approved'`.
- `Integration: approver rejects → document back to 'draft', reason recorded`.
- `Unit: non-approver attempts decide → 403`.

#### 6.2 — PDF & DOCX export
**What**: Render a version to ISO 32000-2 PDF and to DOCX, store in object storage.

**Design**:
- `export-service.render(version, format)`; PDF via `@react-pdf/renderer` from `SowContent` (cover page, sections, pricing table, assumptions, acceptance criteria, AI-disclosure footer per EU AI Act). PDF includes a PAdES-ready signature field placeholder via `pdf-lib`.
- DOCX via `docx`. Files uploaded to S3 (`{org}/{docId}/v{n}.{ext}`); `GET /sows/:id/versions/:n/export?format=pdf|docx` returns a presigned URL. Async render job for large docs.

**Testing**:
- `Integration: render PDF → valid PDF (magic header %PDF-2.0), contains section titles, stored in MinIO, presigned URL resolves`.
- `Integration: render DOCX → opens as valid OOXML, section headings present`.
- `Unit: AI-disclosure footer present when any section aiGenerated`.

#### 6.3 — E-signature integration
**What**: Send the PDF for signature via a provider adapter and track events through webhooks.

**Design**:
```ts
interface ESignPort {
  send(args: { pdf: Buffer; signers: {email:string;name:string}[]; subject:string }):
    Promise<{ externalEnvelopeId: string }>;
  parseWebhook(body: unknown, headers: Record<string,string>): SignatureEvent[];
}
```
- Implementations: DocuSign (JWT-grant OAuth) and Dropbox Sign (API key/OAuth). `POST /sows/:id/sign` creates `signature_request`, transitions document → `client_review`/`out_for_signature`.
- `POST /webhooks/esign/:provider` verifies signature, records `signature_event` rows; on all-signed → document `signed`, emits `sow.signed` CloudEvent. eIDAS: SES baseline; PAdES/LTV when provider supports it.

**Testing**:
- `Integration (mocked DocuSign): POST /sign → external_envelope_id stored, status updated`.
- `Integration: signed webhook (valid signature) → signature_event recorded, document 'signed'`.
- `Integration: webhook with bad signature → 401, no state change`.
- `Unit: parseWebhook maps provider payload → SignatureEvent[]`.

---

## Phase 7: CRM Integration & Lifecycle Eventing

### Purpose
Remove the human handoff: auto-create draft SOWs when a CRM opportunity reaches a configured stage, and notify downstream billing/PM systems on every lifecycle transition using CloudEvents. This delivers the highest-value "CRM-to-SOW" workflow.

### Tasks

#### 7.1 — CRM connection & OAuth
**What**: Connect HubSpot/Salesforce via OAuth 2.0; store encrypted tokens.

**Design**:
- `crm_integration(provider, external_account_id, access_token_encrypted, refresh_token_encrypted, is_active)`. Tokens encrypted at rest (AES-256-GCM, key from env). `GET /integrations/crm/:provider/connect` → OAuth auth-code redirect; callback stores tokens.
- `CrmPort { fetchOpportunity(id), listOpportunitiesByStage(stage), mapToClient(opp) }`.

**Testing**:
- `Integration (mocked OAuth): callback exchanges code → tokens stored encrypted (ciphertext ≠ plaintext)`.
- `Unit: token decrypt round-trips`.
- `Integration: expired token → refresh flow invoked once`.

#### 7.2 — Trigger rules & auto-generation
**What**: When an opportunity reaches `trigger_stage`, create a client/project and enqueue a draft generation.

**Design**:
- `crm_trigger_rule(integration_id, trigger_stage, template_id)`. A `poll-crm` scheduled job (BullMQ repeatable, default every 15 min) lists opportunities at trigger stages; for new ones it upserts `client` (by `crm_external_id`), creates `project`, creates `sow_document`, and enqueues `generate-sow` using the opportunity description as the brief.
- Dedupe by `crm_external_id` to avoid duplicate SOWs.

**Testing**:
- `Integration (mocked CRM): opp at 'Proposal' stage → client+project+draft SOW created, generation job enqueued`.
- `Integration: same opp polled twice → no duplicate document`.
- `Unit: opp below trigger stage → ignored`.

#### 7.3 — CloudEvents lifecycle webhooks
**What**: Emit CloudEvents 1.0 envelopes to subscriber URLs on lifecycle transitions.

**Design**:
- Event types: `sow.created`, `sow.version.generated`, `sow.approved`, `sow.signed`, `sow.rejected`, `sow.scope_creep_detected`. Envelope: `{ specversion:'1.0', type, source:'/sow/{org}', id, time, datacontenttype:'application/json', data }`.
- `webhook_subscription(org_id, url, event_types[], secret)`. `dispatch-webhook` job POSTs with HMAC-SHA256 signature header; retries with exponential backoff; dead-letters after N attempts.

**Testing**:
- `Unit: building envelope for sow.signed → conforms to CloudEvents required attributes`.
- `Integration: signing a SOW → subscriber receives sow.signed with valid HMAC`.
- `Integration: subscriber returns 500 thrice → job retried then dead-lettered`.

---

## Phase 8: Web Application (SPA)

### Purpose
Provide the user-facing editor and dashboards so non-technical consultants can drive everything built in Phases 1-7. After this phase the product is usable without touching the API directly.

### Tasks

#### 8.1 — App shell, auth, generated API client
**What**: React/Vite SPA with auth, routing, and a typed client generated from `openapi.json`.

**Design**: Routes: `/login`, `/dashboard`, `/sows`, `/sows/:id`, `/clauses`, `/rate-cards`, `/templates`, `/settings/integrations`. TanStack Query for server state; auth token in memory + refresh. API client generated via `openapi-typescript`.

**Testing**:
- `E2E (Playwright): login → dashboard renders user's SOW list`.
- `E2E: unauthenticated visit to /sows → redirect to /login`.

#### 8.2 — Brief-to-SOW generation flow
**What**: Brief entry → generate → live job status → rendered draft.

**Design**: Brief form (textarea + vertical select + optional rate card); submit triggers `POST /generate`, polls `/jobs/:id`, then loads the new version into the editor. AI-generated sections visually badged (transparency).

**Testing**:
- `E2E (mocked API): enter brief → progress shown → draft with 7 sections renders, AI badges visible`.
- `E2E: generation failure → error toast, retry available`.

#### 8.3 — Section editor, comments, risk panel, diff & pricing views
**What**: TipTap editor for sections; comment threads; risk-flag side panel; version diff view; pricing table bound to rate cards.

**Design**: Section edits debounce-save via `PUT /sections/:id`. Risk panel lists `riskFlags` with resolve actions. Diff view consumes `GET /diff`. Pricing table edits line items; totals recompute via pricing service on save.

**Testing**:
- `E2E: edit a section, reload → change persisted`.
- `E2E: resolve a risk flag → disappears from open list`.
- `E2E: open diff v1↔v2 → modified sections highlighted`.
- `E2E: add line item with rate-card role → total auto-fills`.

#### 8.4 — Approval & signature UI
**What**: Submit for approval, approver decision screen, send-for-signature, signature status.

**Testing**:
- `E2E: submit for approval → status badge 'internal review'`.
- `E2E (mocked e-sign): send for signature → status 'out for signature', envelope id shown`.

---

## Phase 9: Organisational Memory & Scope-Creep Monitor (AI Differentiators)

### Purpose
Deliver the deep AI-native advantages that incumbents can't easily replicate: learning from the firm's own historical SOWs to recommend clauses/sections, and continuously comparing evolving project communications against the signed SOW to flag scope creep.

### Tasks

#### 9.1 — Historical retrieval (embeddings) for clause/section recommendation
**What**: Embed past clauses and accepted SOW sections; recommend proven wording during drafting.

**Design**:
- Add `pgvector` extension; `clause_embedding(clause_version_id, embedding vector(1536))` and `section_embedding(...)`. On approval/sign, embed bodies via the LLM provider's embeddings endpoint.
- `GET /recommendations?sectionType=&context=` → top-K similar approved sections/clauses by cosine distance, scoped to org (RLS). Generation context (Phase 3) optionally injects top recommendations.
- This is the lightweight realisation of Suggestion 4's knowledge-graph intent without standing up Neo4j; Neo4j remains an optional future swap for relationship-rich recommendation.

**Testing**:
- `Integration (pgvector): embed 3 clauses, query similar to one → that clause ranks first`.
- `Unit: recommendations scoped to requesting org only (no cross-tenant leakage)`.
- `Integration: generation with recommendations on → context includes top sections`.

#### 9.2 — Scope-creep monitor
**What**: Compare new communications (notes/emails) against the signed SOW and raise `sow.scope_creep_detected` events.

**Design**:
- `POST /sows/:id/communications` `{ source, text }` stored relationally. `scope-creep-scan` job (manual or scheduled) runs an LLM diff: "Given the signed SOW and this new communication, list requested items NOT covered by the current scope." Output is a structured list of candidate additions with severity.
- Candidates surfaced in UI and emitted as a CloudEvent; user can convert a candidate into a change-request (new version).

**Testing**:
- `Integration (mocked LLM): communication requesting an out-of-scope deliverable → scope-creep candidate raised + event emitted`.
- `Unit: communication fully within scope → no candidates`.
- `E2E: accept a candidate → new draft version seeded with the addition`.

---

## Phase 10: MCP Server, Hardening & Release

### Purpose
Expose capabilities to AI-agent consumers, harden security to the OWASP API Top 10 baseline, finalise the published OpenAPI + canonical SOW JSON Schema, and package for one-command self-hosting.

### Tasks

#### 10.1 — MCP server
**What**: MCP server exposing SOW tools to agents.

**Design**: `@modelcontextprotocol/sdk` server with tools `generate_sow_draft`, `lookup_rate_card`, `retrieve_past_project`, `list_clause_templates`, `submit_for_approval`, backed by the same `core` services. Auth via API key / OAuth per MCP guidance; all calls run inside `withOrg`.

**Testing**:
- `Integration (MCP test client): list tools → all five present with input schemas`.
- `Integration: call generate_sow_draft → returns SowContent; tenant-scoped`.
- `Integration: missing/invalid credential → tool call rejected`.

#### 10.2 — Security hardening (OWASP API Top 10)
**What**: Systematic controls and tests for the top API risks.

**Design**: Object-level authz checks on every `/:id` route (BOLA), response DTO whitelisting (no excessive data exposure), rate limiting (`@fastify/rate-limit`), input validation everywhere (already via Zod), secrets via env/secret store, RFC 9457 errors that don't leak internals, audit logging on all mutations, encrypted CRM/e-sign tokens. Dependency scanning in CI.

**Testing**:
- `Integration (BOLA): org A token requests org B's SOW by id → 404/403, never the document`.
- `Integration: exceed rate limit → 429 problem+json`.
- `Integration: error responses contain no stack traces in production mode`.
- `Integration: every mutating endpoint writes an audit_log row`.

#### 10.3 — Published specs, packaging & docs
**What**: Commit `openapi.json` and `sow.schema.json` as artifacts; finalise docker-compose; release docs.

**Design**: CI step generates and validates `openapi.json` (Fastify swagger) and `sow.schema.json` (zod-to-json-schema), failing on drift. `docker-compose up` brings up api+worker+web+postgres+redis+minio with migrations auto-applied on api start. Seed script creates a demo org, template, rate card, and clause library.

**Testing**:
- `CI: generated openapi.json validates as OpenAPI 3.1 (spectral lint, 0 errors)`.
- `CI: sow.schema.json validates as JSON Schema 2020-12`.
- `E2E (fresh compose): docker-compose up → register → generate (mocked LLM) → export PDF, full happy path green`.
- `Smoke: seed script populates demo data, dashboard shows it`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, schema, DB+RLS, auth)   ─── required by everything
    │
Phase 2: SOW CRUD, versioning, diff                    ─── requires 1
    │
Phase 3: AI generation engine (CORE)                   ─── requires 1, 2
    ├── Phase 4: Rate cards & pricing                  ─── requires 2 (enhances 3); parallel with 5
    └── Phase 5: Clause library & risk detection       ─── requires 2 (enhances 3); parallel with 4
         │
Phase 6: Approval, export, e-signature                 ─── requires 2 (uses 4/5 output)
    │
Phase 7: CRM integration & CloudEvents                 ─── requires 3, 6
    │
Phase 8: Web SPA                                       ─── requires 2-7 (build incrementally alongside)
    │
Phase 9: Org memory & scope-creep (AI differentiators) ─── requires 3, 5, 6
    │
Phase 10: MCP, hardening, release                      ─── requires all
```

**Parallelism opportunities**
- Phases 4 and 5 can be built concurrently once Phase 2 lands (independent surfaces; both feed Phase 3's generation context).
- Phase 8 (web) can begin against Phase 2 endpoints and grow feature-by-feature in lockstep with Phases 3-7 rather than waiting for all backend work.
- Within Phase 6, PDF/DOCX export (6.2) and e-signature (6.3) are independent until they meet at "send the rendered PDF".
- Phase 10's spec/packaging (10.3) can proceed in parallel with 10.1/10.2.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; new code has tests for happy path and at least the named edge cases.
3. `pnpm lint` and `pnpm format --check` pass with zero errors.
4. `tsc --noEmit` passes across all touched packages (strict mode).
5. New/changed HTTP endpoints appear in the generated `openapi.json`, and CI's spec-drift check is green.
6. Any schema changes have a committed, reviewed drizzle migration; migrations apply cleanly on a fresh database.
7. The feature works end-to-end via the API (and via the SPA where Phase 8 has reached it).
8. New configuration/env vars are added to `.env.example` and documented in the README.
9. New tenant-scoped tables/queries respect RLS (verified by a cross-tenant isolation test).
10. Mutating operations write `audit_log` entries; API errors use RFC 9457 `application/problem+json`.
11. `docker-compose up` still brings the full stack up with migrations applied (no manual steps).
12. Any AI-generated content path tags output as AI-generated (EU AI Act transparency) and records `ai_metadata`.
