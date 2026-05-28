# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Database-as-a-Service Platform · Created: 2026-05-20

## Philosophy

This model treats every state change in the DBaaS control plane as an immutable event appended to a central event store. The current state of any resource (database, branch, endpoint) is derived by replaying its event history. Separate read-optimized materialized views (projections) serve the API and console, while the event store remains the single source of truth.

This pattern is used by AWS internally for CloudTrail event logs, by Kubernetes for its etcd-backed resource versioning, and by financial systems that require complete audit trails. For a DBaaS platform where compliance (SOC 2, HIPAA, GDPR) demands knowing exactly who changed what and when, event sourcing provides this by construction rather than as an afterthought.

The CQRS (Command Query Responsibility Segregation) split means writes go through command handlers that validate business rules and emit events, while reads are served from denormalized projections optimized for specific query patterns. This enables independent scaling of the write path (event ingestion) and read path (API queries, console dashboards).

**Best for:** Platforms where full audit trail is a regulatory requirement, where temporal queries ("what was the state on date X?") are needed, and where the team is comfortable with eventual consistency between writes and reads.

**Trade-offs:**
- Pro: Complete, immutable audit history by construction — no separate audit log needed
- Pro: Temporal queries are natural: replay events to any point in time
- Pro: Write path and read path scale independently
- Pro: New read models can be built from existing events without schema migration
- Pro: Event replay enables debugging and root-cause analysis of any historical state
- Con: Eventual consistency between event store and projections (typically <100ms, but visible)
- Con: Higher implementation complexity; team must understand event sourcing patterns
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: Storage grows linearly with all state changes (snapshotting mitigates but adds complexity)
- Con: Simple CRUD queries require reading from projections, not the source of truth

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 9075:2023 (SQL) | Projections stored in standard relational tables; event store uses JSONB for event payloads |
| CloudEvents v1.0 (CNCF) | Event envelope structure follows CloudEvents spec for interoperability |
| OpenTelemetry | Each event carries a trace_id for distributed tracing correlation |
| OAuth 2.0 (RFC 6749) | Actor identity captured in every event; token scopes validated by command handlers |
| GDPR Article 25 | Right-to-erasure implemented via crypto-shredding: per-org encryption keys deleted to render events unreadable |
| SOC 2 Type II | Immutable event store provides tamper-evident audit trail without additional logging infrastructure |

---

## Event Store (Source of Truth)

```sql
-- The central event store. All state changes flow through here.
-- Events are immutable: no UPDATE or DELETE operations on this table.
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- aggregate root ID (database_id, branch_id, etc.)
    stream_type     VARCHAR(50) NOT NULL,              -- 'database', 'branch', 'endpoint', 'organization', 'project'
    event_type      VARCHAR(100) NOT NULL,             -- 'DatabaseCreated', 'BranchForked', 'EndpointScaled'
    event_version   INTEGER NOT NULL DEFAULT 1,        -- schema version of this event type
    sequence_number BIGINT NOT NULL,                   -- per-stream ordering
    payload         JSONB NOT NULL,                    -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',       -- cross-cutting: actor, ip, request_id, trace_id
    -- metadata example:
    -- {
    --   "actor_id": "uuid",
    --   "actor_type": "user",
    --   "ip_address": "203.0.113.42",
    --   "request_id": "uuid",
    --   "trace_id": "otel-trace-id",
    --   "api_key_id": "uuid"
    -- }
    organization_id UUID NOT NULL,                     -- partition key for multi-tenant isolation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

-- Ordered reads per stream (replay an aggregate's history)
CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);

-- Global ordering for projections that consume all events
CREATE INDEX idx_events_global ON events(created_at, id);

-- Per-organization event queries (audit, compliance)
CREATE INDEX idx_events_org ON events(organization_id, created_at DESC);

-- Event type filtering for specific projectors
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Snapshots for aggregates with long event histories
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,                   -- snapshot is valid up to this sequence
    state           JSONB NOT NULL,                    -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

### Event Type Catalog

```sql
-- Registry of all event types with their JSON schema for validation
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,                    -- JSON Schema Draft 2020-12 for payload validation
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- 'OrganizationCreated', 'OrganizationMemberAdded', 'OrganizationPlanChanged'
-- 'ProjectCreated', 'ProjectDeleted'
-- 'DatabaseProvisioned', 'DatabaseConfigUpdated', 'DatabaseStatusChanged', 'DatabaseDeleted'
-- 'BranchCreated', 'BranchForked', 'BranchProtectionSet', 'BranchDeleted'
-- 'EndpointCreated', 'EndpointScaled', 'EndpointIdled', 'EndpointResumed', 'EndpointDeleted'
-- 'BackupStarted', 'BackupCompleted', 'BackupFailed', 'BackupExpired'
-- 'RestoreStarted', 'RestoreCompleted', 'RestoreFailed'
-- 'DeployRequestOpened', 'DeployRequestApproved', 'DeployRequestDeployed', 'DeployRequestClosed'
-- 'ApiKeyCreated', 'ApiKeyRevoked'
-- 'AIRecommendationGenerated', 'AIRecommendationApplied', 'AIRecommendationDismissed'
```

### Example Event Payloads

```sql
-- DatabaseProvisioned event payload example:
-- {
--   "database_id": "550e8400-e29b-41d4-a716-446655440000",
--   "project_id": "660e8400-e29b-41d4-a716-446655440000",
--   "name": "production-db",
--   "engine": "postgresql",
--   "engine_version": "16",
--   "region_id": "us-east-1",
--   "initial_storage_mb": 1024
-- }

-- EndpointScaled event payload example:
-- {
--   "endpoint_id": "770e8400-e29b-41d4-a716-446655440000",
--   "branch_id": "880e8400-e29b-41d4-a716-446655440000",
--   "previous_cu": 0.25,
--   "new_cu": 2.0,
--   "trigger": "autoscaler",
--   "reason": "cpu_utilization_exceeded_80_pct"
-- }

-- BranchForked event payload example:
-- {
--   "branch_id": "990e8400-e29b-41d4-a716-446655440000",
--   "parent_branch_id": "880e8400-e29b-41d4-a716-446655440000",
--   "name": "feature/add-user-profiles",
--   "parent_lsn": "0/1A2B3C4D",
--   "parent_timestamp": "2026-05-20T10:30:00Z",
--   "git_branch": "feature/add-user-profiles",
--   "pull_request_number": 142
-- }
```

---

## Read Projections (Materialized Views)

These tables are rebuilt from events. They can be dropped and reconstructed at any time.

```sql
-- Current state of organizations (projected from OrganizationCreated, MemberAdded, PlanChanged, etc.)
CREATE TABLE proj_organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_name       VARCHAR(100),
    member_count    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL,
    last_event_seq  BIGINT NOT NULL,                   -- tracks projection position
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Current state of databases
CREATE TABLE proj_databases (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    project_name    VARCHAR(255) NOT NULL,              -- denormalized for read performance
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    engine          VARCHAR(30) NOT NULL,
    engine_version  VARCHAR(20) NOT NULL,
    region_id       VARCHAR(30) NOT NULL,
    region_display  VARCHAR(100) NOT NULL,              -- denormalized
    status          VARCHAR(30) NOT NULL,
    storage_size_mb BIGINT NOT NULL DEFAULT 0,
    branch_count    INTEGER NOT NULL DEFAULT 0,
    endpoint_count  INTEGER NOT NULL DEFAULT 0,
    created_by_email VARCHAR(255),                     -- denormalized
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_db_project ON proj_databases(project_id);
CREATE INDEX idx_proj_db_org ON proj_databases(organization_id);
CREATE INDEX idx_proj_db_status ON proj_databases(status);

-- Current state of branches
CREATE TABLE proj_branches (
    id              UUID PRIMARY KEY,
    database_id     UUID NOT NULL,
    database_name   VARCHAR(255) NOT NULL,              -- denormalized
    name            VARCHAR(255) NOT NULL,
    parent_branch_id UUID,
    parent_branch_name VARCHAR(255),                   -- denormalized
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL,
    endpoint_count  INTEGER NOT NULL DEFAULT 0,
    git_branch      VARCHAR(255),                      -- denormalized from VCS link
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_branches_db ON proj_branches(database_id);

-- Current state of compute endpoints
CREATE TABLE proj_endpoints (
    id              UUID PRIMARY KEY,
    branch_id       UUID NOT NULL,
    database_id     UUID NOT NULL,                     -- denormalized for direct lookups
    host            VARCHAR(255) NOT NULL,
    port            INTEGER NOT NULL,
    type            VARCHAR(20) NOT NULL,
    min_cu          NUMERIC(5,2) NOT NULL,
    max_cu          NUMERIC(5,2) NOT NULL,
    current_cu      NUMERIC(5,2) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    last_active_at  TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_endpoints_branch ON proj_endpoints(branch_id);
CREATE INDEX idx_proj_endpoints_status ON proj_endpoints(status);

-- Usage aggregation projection (pre-computed for billing)
CREATE TABLE proj_usage_summary (
    organization_id UUID NOT NULL,
    project_id      UUID NOT NULL,
    period          DATE NOT NULL,                     -- first day of billing period
    compute_hours   NUMERIC(20,6) NOT NULL DEFAULT 0,
    storage_gb_hours NUMERIC(20,6) NOT NULL DEFAULT 0,
    rows_read       BIGINT NOT NULL DEFAULT 0,
    rows_written    BIGINT NOT NULL DEFAULT 0,
    data_transfer_gb NUMERIC(20,6) NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organization_id, project_id, period)
);

-- AI recommendations projection
CREATE TABLE proj_ai_recommendations (
    id              UUID PRIMARY KEY,
    database_id     UUID NOT NULL,
    branch_id       UUID,
    type            VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    suggested_sql   TEXT,
    estimated_impact JSONB,
    status          VARCHAR(20) NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_ai_recs ON proj_ai_recommendations(database_id, status);
```

## Projection Tracking

```sql
-- Tracks each projector's position in the event stream
CREATE TABLE projection_checkpoints (
    projector_name  VARCHAR(100) PRIMARY KEY,          -- 'databases', 'branches', 'usage_summary'
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    last_processed  TIMESTAMPTZ NOT NULL DEFAULT now(),
    events_behind   BIGINT NOT NULL DEFAULT 0
);
```

## Command-Side Reference Data

```sql
-- These tables support command validation and are NOT event-sourced
CREATE TABLE regions (
    id              VARCHAR(30) PRIMARY KEY,
    cloud_provider  VARCHAR(20) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    max_projects    INTEGER,
    max_databases   INTEGER,
    max_branches    INTEGER,
    max_compute_cu  NUMERIC(10,2),
    backup_retention_days INTEGER NOT NULL DEFAULT 7,
    price_monthly_cents INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,
    description     TEXT
);
```

---

## Temporal Query Examples

```sql
-- What was the state of database X at a specific point in time?
-- Replay events up to that timestamp:
SELECT payload
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'database'
  AND created_at <= '2026-05-15T14:30:00Z'
ORDER BY sequence_number ASC;

-- Who made changes to branch Y in the last 24 hours?
SELECT event_type,
       metadata->>'actor_id' AS actor,
       metadata->>'ip_address' AS ip,
       payload,
       created_at
FROM events
WHERE stream_id = '880e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'branch'
  AND created_at > now() - INTERVAL '24 hours'
ORDER BY sequence_number ASC;

-- Complete audit trail for an organization (SOC 2 / GDPR compliance)
SELECT event_type, stream_type, stream_id,
       metadata->>'actor_id' AS actor,
       metadata->>'actor_type' AS actor_type,
       payload,
       created_at
FROM events
WHERE organization_id = '110e8400-e29b-41d4-a716-446655440000'
  AND created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY created_at ASC;

-- How many databases were created per day last month? (analytics from events)
SELECT date_trunc('day', created_at) AS day,
       COUNT(*) AS databases_created
FROM events
WHERE event_type = 'DatabaseProvisioned'
  AND created_at >= '2026-04-01'
  AND created_at < '2026-05-01'
GROUP BY 1
ORDER BY 1;
```

---

## GDPR Crypto-Shredding

```sql
-- Per-organization encryption keys for GDPR right-to-erasure
CREATE TABLE org_encryption_keys (
    organization_id UUID PRIMARY KEY,
    key_id          VARCHAR(100) NOT NULL,             -- reference to KMS key
    algorithm       VARCHAR(20) NOT NULL DEFAULT 'AES-256-GCM',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at      TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ                        -- when set, org's events are unreadable
);

-- To "erase" an organization's data under GDPR:
-- 1. DELETE FROM org_encryption_keys WHERE organization_id = '...';
-- 2. Events remain in the store but their encrypted payload fields are unreadable
-- 3. Projections for that org are purged
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Event Registry | 1 | event_type_registry |
| Projections | 6 | proj_organizations, proj_databases, proj_branches, proj_endpoints, proj_usage_summary, proj_ai_recommendations |
| Projection Infrastructure | 1 | projection_checkpoints |
| Reference Data | 2 | regions, plans |
| GDPR | 1 | org_encryption_keys |
| **Total** | **13** | Plus any additional projections as read requirements grow |

---

## Key Design Decisions

1. **Single `events` table as the source of truth** — all state changes flow through one append-only table. This eliminates the need for separate audit logging and guarantees that the audit trail is complete by construction.

2. **CloudEvents-aligned metadata envelope** — every event carries actor identity, IP address, request ID, and OpenTelemetry trace ID in a standardized metadata structure, enabling correlation across observability systems.

3. **Stream-level optimistic concurrency** — the `UNIQUE(stream_id, sequence_number)` constraint prevents conflicting writes to the same aggregate. Command handlers read the current sequence number before appending.

4. **Projections are disposable** — all `proj_*` tables can be dropped and rebuilt from the event store. This enables schema evolution of read models without data migration.

5. **Snapshots for performance** — aggregates with long histories (e.g., a production database with thousands of scaling events) use snapshots to avoid replaying the full event stream on every command.

6. **GDPR via crypto-shredding** — rather than deleting events (which would break the immutable log), organization-level encryption keys are destroyed, rendering the encrypted portions of event payloads unreadable.

7. **Denormalized projections** — read-side tables include redundant data (e.g., `project_name` in `proj_databases`) to avoid joins at query time. This is the CQRS trade-off: storage for speed.

8. **Projection checkpoints for exactly-once processing** — each projector tracks its position in the event stream, enabling restart without reprocessing and monitoring of projection lag.

9. **Fewer tables overall (13 vs ~26)** — the event-sourced model has roughly half the tables of the normalized model because many distinct entity tables collapse into event streams and projections.

10. **Event versioning via `event_version` field** — as event schemas evolve, the version field enables upcasters to transform old event formats to new ones during replay, avoiding event store migrations.
