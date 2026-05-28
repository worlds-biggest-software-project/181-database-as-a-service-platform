# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Database-as-a-Service Platform · Created: 2026-05-20

## Philosophy

This model uses traditional relational tables for the core resource hierarchy (organizations, projects, databases, branches, endpoints) but delegates variable, engine-specific, region-specific, and rapidly-evolving fields to JSONB columns. The key insight is that a DBaaS platform must support multiple database engines (PostgreSQL, MySQL, and potentially others), multiple cloud providers, and rapidly evolving features — all of which introduce schema variability that normalized tables handle poorly.

This is the pattern used by Stripe's internal data model (core fields relational, metadata JSONB), by Shopify's multi-tenant platform, and by many control planes that must absorb new configuration dimensions without DDL migrations. PostgreSQL's JSONB type provides indexing (GIN indexes), containment queries (`@>`), and jsonpath expressions, making it nearly as queryable as relational columns for well-known paths.

The hybrid approach reduces table count compared to pure normalization (no separate tables for engine-specific configs, provider-specific settings, or feature flags), accelerates MVP development (add a new field to JSONB without a migration), and still provides relational integrity for the core resource graph.

**Best for:** Teams building an MVP that must support multiple engines and cloud providers from early on, where schema flexibility is valued over strict normalization, and where the development pace demands minimal DDL migrations.

**Trade-offs:**
- Pro: Fewer tables; engine/provider-specific fields don't need separate tables
- Pro: Adding new configuration fields requires no schema migration
- Pro: JSONB columns are fully indexable with GIN indexes and jsonpath
- Pro: API responses can often serve JSONB columns directly without transformation
- Pro: Multi-engine and multi-cloud variability absorbed without schema explosion
- Con: JSONB fields lack database-enforced constraints (CHECK, NOT NULL, FK)
- Con: Type safety for JSONB contents must be enforced in application code or JSON Schema validation
- Con: GIN indexes are larger and slower to update than B-tree indexes on dedicated columns
- Con: Complex queries on deeply nested JSONB paths can be less readable than joins
- Con: Schema documentation must be maintained separately (JSONB structure isn't visible in pg_catalog)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 9075:2023 (SQL) | SQL/JSON functions (JSON_TABLE, JSON_QUERY) from SQL:2023 used for JSONB querying |
| JSON Schema 2020-12 | JSONB column structures documented and validated against JSON Schema definitions |
| OpenAPI 3.1 | API request/response schemas align with JSON Schema, which maps directly to JSONB column schemas |
| OAuth 2.0 (RFC 6749) | Auth config stored in JSONB for flexibility across providers (GitHub, GitLab, SAML, OIDC) |
| ISO 3166-1/2 | Region data embedded in JSONB with country_code field validated against ISO 3166 |
| OpenTelemetry | Observability configuration stored in JSONB per project for flexible export targets |

---

## Core Resource Hierarchy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'deleted')),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "billing_email": "billing@acme.com",
    --   "default_region": "us-east-1",
    --   "sso_config": {
    --     "provider": "okta",
    --     "domain": "acme.okta.com",
    --     "client_id": "...",
    --     "protocol": "saml"
    --   },
    --   "compliance": {
    --     "data_residency": ["us", "eu"],
    --     "require_encryption": true,
    --     "audit_log_retention_days": 365
    --   },
    --   "feature_flags": {
    --     "ai_recommendations": true,
    --     "multi_engine": false,
    --     "branching_v2": true
    --   }
    -- }
    plan            JSONB NOT NULL DEFAULT '{}',
    -- plan example:
    -- {
    --   "name": "pro",
    --   "max_projects": 25,
    --   "max_databases": 100,
    --   "max_branches_per_db": 50,
    --   "max_compute_cu": 8.0,
    --   "backup_retention_days": 14,
    --   "pitr_enabled": true,
    --   "price_monthly_cents": 2500
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_slug ON organizations(slug);
CREATE INDEX idx_org_settings ON organizations USING GIN (settings jsonb_path_ops);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "external_id": "okta-user-123",
    --   "idp_provider": "okta",
    --   "preferences": {"theme": "dark", "timezone": "America/New_York"},
    --   "mfa_enabled": true
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(30) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example (overrides for specific resources):
    -- {
    --   "project_access": {
    --     "proj-uuid-1": "admin",
    --     "proj-uuid-2": "viewer"
    --   },
    --   "can_manage_billing": false,
    --   "can_invite_members": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_orgmem_org ON organization_members(organization_id);
CREATE INDEX idx_orgmem_user ON organization_members(user_id);
```

## Projects and Databases

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'paused', 'deleted')),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_region": "eu-west-1",
    --   "default_engine": "postgresql",
    --   "observability": {
    --     "prometheus_endpoint": "https://prom.acme.com/push",
    --     "otel_endpoint": "https://otel.acme.com:4317",
    --     "log_level": "info"
    --   },
    --   "vcs_integration": {
    --     "provider": "github",
    --     "repo_owner": "acme",
    --     "repo_name": "backend",
    --     "installation_id": "12345",
    --     "auto_branch": true,
    --     "auto_delete_on_merge": true
    --   },
    --   "ip_allowlist": ["203.0.113.0/24", "198.51.100.0/24"]
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_projects_settings ON projects USING GIN (settings jsonb_path_ops);

CREATE TABLE databases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    engine          VARCHAR(30) NOT NULL DEFAULT 'postgresql'
                    CHECK (engine IN ('postgresql', 'mysql')),
    engine_version  VARCHAR(20) NOT NULL,
    region          VARCHAR(30) NOT NULL,              -- 'us-east-1', 'eu-west-1'
    cloud_provider  VARCHAR(20) NOT NULL DEFAULT 'aws',
    status          VARCHAR(30) NOT NULL DEFAULT 'provisioning'
                    CHECK (status IN ('provisioning', 'available', 'scaling',
                                      'suspended', 'deleting', 'deleted', 'error')),
    storage_size_mb BIGINT NOT NULL DEFAULT 0,
    engine_config   JSONB NOT NULL DEFAULT '{}',
    -- engine_config example (PostgreSQL):
    -- {
    --   "max_connections": 100,
    --   "shared_buffers": "256MB",
    --   "work_mem": "4MB",
    --   "maintenance_work_mem": "64MB",
    --   "effective_cache_size": "768MB",
    --   "wal_level": "replica",
    --   "max_wal_senders": 10,
    --   "extensions_enabled": ["pgvector", "postgis", "pg_trgm"],
    --   "pg_bouncer": {
    --     "pool_mode": "transaction",
    --     "max_client_connections": 200,
    --     "default_pool_size": 25
    --   }
    -- }
    -- engine_config example (MySQL):
    -- {
    --   "max_connections": 151,
    --   "innodb_buffer_pool_size": "256M",
    --   "innodb_flush_log_at_trx_commit": 1,
    --   "binlog_format": "ROW",
    --   "character_set_server": "utf8mb4"
    -- }
    security        JSONB NOT NULL DEFAULT '{}',
    -- security example:
    -- {
    --   "encryption_at_rest": true,
    --   "encryption_algorithm": "AES-256",
    --   "kms_key_id": "arn:aws:kms:...",
    --   "min_tls_version": "1.3",
    --   "require_ssl": true,
    --   "ip_allowlist": ["10.0.0.0/8"]
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_db_project ON databases(project_id);
CREATE INDEX idx_db_status ON databases(status) WHERE status != 'deleted';
CREATE INDEX idx_db_engine_config ON databases USING GIN (engine_config jsonb_path_ops);
```

## Branches and Compute Endpoints

```sql
CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    parent_branch_id UUID REFERENCES branches(id),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'ready', 'deleting', 'deleted', 'error')),
    branch_point    JSONB,
    -- branch_point example:
    -- {
    --   "parent_lsn": "0/1A2B3C4D",
    --   "parent_timestamp": "2026-05-20T10:30:00Z",
    --   "cow_storage_path": "s3://neon-storage/branches/...",
    --   "data_size_at_fork_mb": 2048
    -- }
    vcs_link        JSONB,
    -- vcs_link example:
    -- {
    --   "git_branch": "feature/user-profiles",
    --   "pull_request_number": 142,
    --   "pull_request_url": "https://github.com/acme/backend/pull/142",
    --   "auto_delete_on_merge": true
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_branches_db ON branches(database_id);
CREATE UNIQUE INDEX idx_branches_default ON branches(database_id) WHERE is_default = true;

CREATE TABLE compute_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    host            VARCHAR(255) NOT NULL UNIQUE,
    port            INTEGER NOT NULL DEFAULT 5432,
    type            VARCHAR(20) NOT NULL DEFAULT 'read_write'
                    CHECK (type IN ('read_write', 'read_only')),
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'active', 'idle', 'suspended', 'error')),
    compute         JSONB NOT NULL DEFAULT '{}',
    -- compute example:
    -- {
    --   "min_cu": 0.25,
    --   "max_cu": 4.0,
    --   "current_cu": 1.0,
    --   "autoscaling": true,
    --   "scale_to_zero": true,
    --   "idle_timeout_seconds": 300,
    --   "suspend_timeout_seconds": 3600,
    --   "provisioner": "k8s",
    --   "instance_type": "c6g.medium",
    --   "availability_zone": "us-east-1a"
    -- }
    connection_pooling JSONB,
    -- connection_pooling example:
    -- {
    --   "enabled": true,
    --   "pool_mode": "transaction",
    --   "pooler_host": "pooler-us-east-1.example.com",
    --   "pooler_port": 6432,
    --   "max_client_connections": 200,
    --   "default_pool_size": 25
    -- }
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_endpoints_branch ON compute_endpoints(branch_id);
CREATE INDEX idx_endpoints_status ON compute_endpoints(status);
CREATE INDEX idx_endpoints_compute ON compute_endpoints USING GIN (compute jsonb_path_ops);
```

## Access Control and API Keys

```sql
CREATE TABLE database_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    name            VARCHAR(63) NOT NULL,
    password_hash   TEXT,
    privileges      JSONB NOT NULL DEFAULT '{}',
    -- privileges example:
    -- {
    --   "is_superuser": false,
    --   "can_login": true,
    --   "can_create_db": false,
    --   "connection_limit": 20,
    --   "valid_until": "2027-01-01T00:00:00Z",
    --   "granted_roles": ["readonly", "app_user"],
    --   "rls_policies": ["tenant_isolation"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(database_id, name)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(10) NOT NULL,
    key_hash        TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "scopes": ["projects:read", "databases:write", "branches:*"],
    --   "ip_restrictions": ["203.0.113.0/24"],
    --   "rate_limit_rpm": 1000,
    --   "allowed_projects": ["proj-uuid-1", "proj-uuid-2"]
    -- }
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

## Backups and Operations

```sql
CREATE TABLE backups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    type            VARCHAR(20) NOT NULL
                    CHECK (type IN ('scheduled', 'manual', 'pre_migration')),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                    CHECK (status IN ('in_progress', 'completed', 'failed', 'expired')),
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "storage_path": "s3://backups/org-123/db-456/2026-05-20.tar.gz",
    --   "size_bytes": 1073741824,
    --   "wal_position": "0/1A2B3C4D",
    --   "compression": "zstd",
    --   "encryption_key_id": "kms-key-789",
    --   "pg_version": "16.3",
    --   "duration_seconds": 45,
    --   "tables_count": 127,
    --   "error_message": null
    -- }
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_backups_branch ON backups(branch_id, created_at DESC);

CREATE TABLE operations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    operation_type  VARCHAR(50) NOT NULL,              -- 'provision', 'scale', 'backup', 'restore', 'migrate', 'delete'
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    input           JSONB NOT NULL DEFAULT '{}',
    output          JSONB,
    progress_pct    INTEGER DEFAULT 0,
    started_by      UUID REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error           JSONB,
    -- error example:
    -- {
    --   "code": "INSUFFICIENT_CAPACITY",
    --   "message": "No available compute nodes in us-east-1a",
    --   "retryable": true,
    --   "retry_after": "2026-05-20T11:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ops_resource ON operations(resource_type, resource_id, created_at DESC);
CREATE INDEX idx_ops_status ON operations(status) WHERE status IN ('pending', 'running');
CREATE INDEX idx_ops_org ON operations(organization_id, created_at DESC);
```

## Audit Log and Usage

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "request_id": "req-uuid",
    --   "api_key_id": "key-uuid",
    --   "changes": {
    --     "before": {"status": "idle", "current_cu": 0},
    --     "after": {"status": "active", "current_cu": 2.0}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE audit_log_2026_06 PARTITION OF audit_log
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);

CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    project_id      UUID NOT NULL,
    database_id     UUID,
    metrics         JSONB NOT NULL,
    -- metrics example:
    -- {
    --   "compute_hours": 24.5,
    --   "storage_bytes": 10737418240,
    --   "rows_read": 1500000,
    --   "rows_written": 250000,
    --   "data_transfer_bytes": 536870912,
    --   "active_endpoints": 3,
    --   "branches_count": 7
    -- }
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (period_start);

CREATE INDEX idx_usage_org_period ON usage_records(organization_id, period_start);
```

## AI Features

```sql
CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    branch_id       UUID REFERENCES branches(id),
    type            VARCHAR(30) NOT NULL
                    CHECK (type IN ('index', 'schema', 'query_rewrite', 'scaling', 'cost')),
    severity        VARCHAR(10) NOT NULL DEFAULT 'info',
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'accepted', 'dismissed', 'applied', 'expired')),
    recommendation  JSONB NOT NULL,
    -- recommendation example (index suggestion):
    -- {
    --   "description": "Adding a composite index on (tenant_id, created_at) would reduce scan time by ~60%",
    --   "suggested_sql": "CREATE INDEX CONCURRENTLY idx_orders_tenant_created ON orders(tenant_id, created_at DESC);",
    --   "estimated_impact": {
    --     "query_time_reduction_pct": 60,
    --     "affected_queries": 12,
    --     "storage_increase_mb": 45
    --   },
    --   "analysis": {
    --     "sample_queries": ["SELECT * FROM orders WHERE tenant_id = $1 ORDER BY created_at DESC LIMIT 20"],
    --     "current_plan": "Seq Scan on orders (cost=0.00..18234.00 rows=500 width=128)",
    --     "projected_plan": "Index Scan using idx_orders_tenant_created (cost=0.43..24.50 rows=500 width=128)"
    --   }
    -- }
    applied_by      UUID REFERENCES users(id),
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_recs_db ON ai_recommendations(database_id, status);
```

---

## JSONB Query Examples

```sql
-- Find all databases with pgvector extension enabled
SELECT id, name, engine_version
FROM databases
WHERE engine_config @> '{"extensions_enabled": ["pgvector"]}';

-- Find organizations with SSO configured via Okta
SELECT id, name, settings->'sso_config'->>'domain' AS okta_domain
FROM organizations
WHERE settings @> '{"sso_config": {"provider": "okta"}}';

-- Find endpoints currently using more than 2 compute units
SELECT e.id, e.host, e.compute->>'current_cu' AS current_cu,
       b.name AS branch_name, d.name AS database_name
FROM compute_endpoints e
JOIN branches b ON e.branch_id = b.id
JOIN databases d ON b.database_id = d.id
WHERE (e.compute->>'current_cu')::numeric > 2.0
  AND e.status = 'active';

-- Aggregate usage metrics across an organization for billing
SELECT project_id,
       SUM((metrics->>'compute_hours')::numeric) AS total_compute_hours,
       SUM((metrics->>'storage_bytes')::bigint) AS total_storage_bytes,
       SUM((metrics->>'rows_read')::bigint) AS total_rows_read
FROM usage_records
WHERE organization_id = '110e8400-e29b-41d4-a716-446655440000'
  AND period_start >= '2026-05-01'
  AND period_start < '2026-06-01'
GROUP BY project_id;

-- Find all branches linked to open pull requests
SELECT b.id, b.name, b.vcs_link->>'pull_request_url' AS pr_url,
       d.name AS database_name
FROM branches b
JOIN databases d ON b.database_id = d.id
WHERE b.vcs_link ? 'pull_request_number'
  AND b.status = 'ready';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Organization | 3 | organizations, users, organization_members |
| Projects & Databases | 2 | projects, databases |
| Branching & Compute | 2 | branches, compute_endpoints |
| Access Control | 2 | database_roles, api_keys |
| Backups & Operations | 2 | backups, operations |
| Audit & Usage | 2 | audit_log (partitioned), usage_records (partitioned) |
| AI Features | 1 | ai_recommendations |
| **Total** | **14** | JSONB absorbs what would be ~12 additional tables in the normalized model |

---

## Key Design Decisions

1. **JSONB for variability, relational for identity** — core resource IDs, foreign keys, and lifecycle status are relational columns with constraints. Everything that varies by engine, provider, or feature release goes into JSONB.

2. **No separate `plans` table** — plan details are embedded in the organization's `plan` JSONB column. This avoids the complexity of plan versioning tables and supports custom enterprise plans as arbitrary JSON.

3. **VCS integration embedded in branches** — the `vcs_link` JSONB column on branches eliminates the need for separate `vcs_integrations` and `vcs_branch_links` tables, with project-level VCS config in `projects.settings`.

4. **Engine configuration as JSONB** — PostgreSQL GUC parameters and MySQL system variables have completely different schemas. JSONB absorbs this without engine-specific configuration tables.

5. **Partitioned audit log and usage records** — high-volume tables are partitioned by time for efficient retention management (drop old partitions) and query performance.

6. **Operations table as a unified job tracker** — instead of separate tables for restore operations, migration operations, and scaling operations, a single `operations` table with JSONB input/output handles all async operations.

7. **GIN indexes on JSONB columns** — `jsonb_path_ops` GIN indexes enable efficient containment queries (`@>`) for finding databases by extension, organizations by SSO provider, etc.

8. **Security config per database** — encryption, TLS, and IP allowlist settings are embedded in the database's `security` JSONB rather than separate tables, keeping security configuration co-located with the resource.

9. **14 tables vs 26 in normalized model** — JSONB absorbs approximately 12 tables worth of variability (configurations, VCS links, pool settings, plan details, permissions overrides).

10. **JSON Schema validation at the application layer** — each JSONB column has a documented JSON Schema (aligned with JSON Schema 2020-12 / OpenAPI 3.1) that is validated on write by the API layer, compensating for the lack of database-level constraints on JSONB contents.
