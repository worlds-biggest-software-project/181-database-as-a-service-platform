# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Database-as-a-Service Platform · Created: 2026-05-20

## Philosophy

This model follows classical normalized relational design where every concept in the DBaaS control plane has its own dedicated table with strict foreign key relationships. Organizations, projects, database clusters, branches, compute endpoints, backups, roles, API keys, and billing records are all first-class entities with well-defined referential integrity.

The approach mirrors how AWS RDS, Neon, and Supabase structure their management APIs: resources are hierarchical (organization > project > database > branch > endpoint), and each resource has a stable identity (UUID), lifecycle state, and audit timestamps. Every relationship is explicit via foreign keys, making the schema self-documenting and queryable without application-layer knowledge.

This is the most straightforward approach for a team experienced with PostgreSQL. It prioritizes data integrity, query flexibility, and tooling compatibility (ORMs, migration frameworks, BI tools all work naturally with normalized schemas).

**Best for:** Teams that value data integrity, need complex cross-entity reporting, and operate in regulated environments where auditability of the schema itself matters.

**Trade-offs:**
- Pro: Maximum referential integrity; database enforces business rules
- Pro: Standard PostgreSQL tooling, ORMs, and migration frameworks work out of the box
- Pro: Easy to reason about; schema is self-documenting
- Pro: Efficient for complex joins and cross-entity queries
- Con: More tables to manage (~45-55 tables for full scope)
- Con: Schema migrations for new features require DDL changes and coordination
- Con: Jurisdiction-specific or engine-specific fields require either nullable columns or additional junction tables
- Con: Historical state queries require separate audit/history tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 9075:2023 (SQL) | Full SQL compliance for DDL and queries; leverages TIMESTAMPTZ, UUID, JSONB where appropriate |
| OAuth 2.0 (RFC 6749) | API key and token tables model Client Credentials grant; token expiry and scopes stored relationally |
| OpenTelemetry | Metrics and traces reference database/branch/endpoint IDs via foreign keys for correlation |
| ISO 3166-1/2 | Region codes stored as CHAR(2)/VARCHAR(6) referencing a regions lookup table |
| SCIM 2.0 (RFC 7643) | User provisioning modeled via organization_members with external_id for IdP sync |
| TLS 1.3 (RFC 8446) | Connection encryption settings stored per endpoint with minimum TLS version constraints |
| OpenAPI 3.1 | API key scopes and permissions modeled to match OpenAPI security scheme definitions |

---

## Identity and Organization Management

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_id         UUID REFERENCES plans(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'deleted')),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_status ON organizations(status);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    external_id     VARCHAR(255),          -- SCIM external ID for IdP sync
    idp_provider    VARCHAR(50),           -- 'google', 'github', 'okta', 'saml'
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'deleted')),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_external_id ON users(external_id) WHERE external_id IS NOT NULL;

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(30) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    invited_by      UUID REFERENCES users(id),
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

## Plans and Billing

```sql
CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,          -- 'free', 'pro', 'team', 'enterprise'
    display_name    VARCHAR(255) NOT NULL,
    max_projects    INTEGER,                         -- NULL = unlimited
    max_databases   INTEGER,
    max_branches    INTEGER,
    max_compute_cu  NUMERIC(10,2),                  -- max compute units
    backup_retention_days INTEGER NOT NULL DEFAULT 7,
    pitr_enabled    BOOLEAN NOT NULL DEFAULT false,
    price_monthly_cents INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE billing_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    stripe_customer_id VARCHAR(255),
    payment_method  VARCHAR(50),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    billing_cycle   VARCHAR(10) NOT NULL DEFAULT 'monthly'
                    CHECK (billing_cycle IN ('monthly', 'annual')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    project_id      UUID NOT NULL REFERENCES projects(id),
    database_id     UUID REFERENCES databases(id),
    metric_type     VARCHAR(50) NOT NULL,            -- 'compute_hours', 'storage_bytes', 'rows_read', 'rows_written', 'data_transfer_bytes'
    quantity        NUMERIC(20,6) NOT NULL,
    unit            VARCHAR(30) NOT NULL,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_usage_org_period ON usage_records(organization_id, period_start, period_end);
CREATE INDEX idx_usage_project ON usage_records(project_id, metric_type, period_start);
```

## Project and Database Management

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    default_region  VARCHAR(20) NOT NULL,             -- e.g., 'us-east-1', 'eu-west-1'
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'paused', 'deleted')),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE INDEX idx_projects_org ON projects(organization_id);

CREATE TABLE regions (
    id              VARCHAR(30) PRIMARY KEY,          -- 'us-east-1', 'eu-west-1'
    cloud_provider  VARCHAR(20) NOT NULL,             -- 'aws', 'gcp', 'azure'
    display_name    VARCHAR(100) NOT NULL,
    country_code    CHAR(2) NOT NULL,                 -- ISO 3166-1 alpha-2
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE databases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    engine          VARCHAR(30) NOT NULL DEFAULT 'postgresql'
                    CHECK (engine IN ('postgresql', 'mysql')),
    engine_version  VARCHAR(20) NOT NULL,             -- '16', '17', '8.0'
    region_id       VARCHAR(30) NOT NULL REFERENCES regions(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'provisioning'
                    CHECK (status IN ('provisioning', 'available', 'scaling', 'suspended', 'deleting', 'deleted', 'error')),
    storage_size_mb BIGINT NOT NULL DEFAULT 0,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_databases_project ON databases(project_id);
CREATE INDEX idx_databases_status ON databases(status) WHERE status != 'deleted';

CREATE TABLE database_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    parameter_name  VARCHAR(255) NOT NULL,            -- PostgreSQL GUC parameters
    parameter_value TEXT NOT NULL,
    is_restart_required BOOLEAN NOT NULL DEFAULT false,
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(database_id, parameter_name)
);
```

## Branching and Compute

```sql
CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    parent_branch_id UUID REFERENCES branches(id),
    parent_lsn      VARCHAR(30),                      -- WAL Log Sequence Number at branch point
    parent_timestamp TIMESTAMPTZ,                     -- timestamp at branch point
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'ready', 'deleting', 'deleted', 'error')),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_branches_database ON branches(database_id);
CREATE INDEX idx_branches_parent ON branches(parent_branch_id) WHERE parent_branch_id IS NOT NULL;
CREATE UNIQUE INDEX idx_branches_default ON branches(database_id) WHERE is_default = true;

CREATE TABLE compute_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    host            VARCHAR(255) NOT NULL UNIQUE,      -- connection hostname
    port            INTEGER NOT NULL DEFAULT 5432,
    type            VARCHAR(20) NOT NULL DEFAULT 'read_write'
                    CHECK (type IN ('read_write', 'read_only')),
    min_cu          NUMERIC(5,2) NOT NULL DEFAULT 0.25,  -- minimum compute units
    max_cu          NUMERIC(5,2) NOT NULL DEFAULT 4.0,   -- maximum compute units
    current_cu      NUMERIC(5,2) NOT NULL DEFAULT 0,
    autoscaling     BOOLEAN NOT NULL DEFAULT true,
    scale_to_zero   BOOLEAN NOT NULL DEFAULT true,
    idle_timeout_seconds INTEGER NOT NULL DEFAULT 300,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating'
                    CHECK (status IN ('creating', 'active', 'idle', 'suspended', 'error')),
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_endpoints_branch ON compute_endpoints(branch_id);
CREATE INDEX idx_endpoints_status ON compute_endpoints(status);
```

## Connection Pooling and Access Control

```sql
CREATE TABLE connection_pools (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES compute_endpoints(id) ON DELETE CASCADE,
    pool_mode       VARCHAR(20) NOT NULL DEFAULT 'transaction'
                    CHECK (pool_mode IN ('session', 'transaction', 'statement')),
    max_client_connections INTEGER NOT NULL DEFAULT 100,
    default_pool_size INTEGER NOT NULL DEFAULT 15,
    host            VARCHAR(255) NOT NULL,
    port            INTEGER NOT NULL DEFAULT 6432,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE database_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    name            VARCHAR(63) NOT NULL,              -- PostgreSQL role name
    password_hash   TEXT,
    is_superuser    BOOLEAN NOT NULL DEFAULT false,
    can_login       BOOLEAN NOT NULL DEFAULT true,
    can_create_db   BOOLEAN NOT NULL DEFAULT false,
    connection_limit INTEGER DEFAULT -1,               -- -1 = unlimited
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(database_id, name)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(10) NOT NULL,              -- first 8 chars for identification
    key_hash        TEXT NOT NULL,                      -- bcrypt hash of the full key
    scopes          TEXT[] NOT NULL DEFAULT '{}',       -- ['projects:read', 'databases:write']
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

## Backups and Recovery

```sql
CREATE TABLE backups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    type            VARCHAR(20) NOT NULL
                    CHECK (type IN ('scheduled', 'manual', 'pre_migration')),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                    CHECK (status IN ('in_progress', 'completed', 'failed', 'expired', 'deleted')),
    storage_path    TEXT,
    size_bytes      BIGINT,
    wal_position    VARCHAR(30),                       -- WAL LSN at backup time
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_backups_branch ON backups(branch_id, created_at DESC);
CREATE INDEX idx_backups_status ON backups(status) WHERE status = 'in_progress';
CREATE INDEX idx_backups_expires ON backups(expires_at) WHERE expires_at IS NOT NULL;

CREATE TABLE restore_operations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_backup_id UUID REFERENCES backups(id),
    source_branch_id UUID REFERENCES branches(id),
    target_branch_id UUID NOT NULL REFERENCES branches(id),
    restore_type    VARCHAR(20) NOT NULL
                    CHECK (restore_type IN ('full', 'pitr')),
    pitr_target     TIMESTAMPTZ,                       -- for point-in-time recovery
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'restoring', 'completed', 'failed')),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Schema Migrations and Deploy Requests

```sql
CREATE TABLE migration_histories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    sql_up          TEXT NOT NULL,
    sql_down        TEXT,
    checksum        VARCHAR(64) NOT NULL,              -- SHA-256 of sql_up
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    applied_by      UUID REFERENCES users(id),
    execution_time_ms INTEGER
);

CREATE INDEX idx_migrations_branch ON migration_histories(branch_id, version);

CREATE TABLE deploy_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    source_branch_id UUID NOT NULL REFERENCES branches(id),
    target_branch_id UUID NOT NULL REFERENCES branches(id),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'approved', 'deploying', 'deployed', 'closed', 'failed')),
    ddl_statements  TEXT[],
    is_non_blocking BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES users(id),
    reviewed_by     UUID REFERENCES users(id),
    deployed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deploy_requests_db ON deploy_requests(database_id, status);
```

## Observability and Monitoring

```sql
CREATE TABLE metric_definitions (
    id              VARCHAR(100) PRIMARY KEY,          -- 'cpu_utilization', 'storage_bytes', 'connections_active'
    display_name    VARCHAR(255) NOT NULL,
    unit            VARCHAR(30) NOT NULL,              -- 'percent', 'bytes', 'count', 'milliseconds'
    description     TEXT,
    otel_metric_name VARCHAR(255)                      -- OpenTelemetry metric name mapping
);

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    metric_id       VARCHAR(100) NOT NULL REFERENCES metric_definitions(id),
    condition       VARCHAR(10) NOT NULL               -- 'gt', 'lt', 'gte', 'lte', 'eq'
                    CHECK (condition IN ('gt', 'lt', 'gte', 'lte', 'eq')),
    threshold       NUMERIC NOT NULL,
    evaluation_period_seconds INTEGER NOT NULL DEFAULT 300,
    notification_channels JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "email", "target": "ops@example.com"}, {"type": "slack", "webhook": "..."}]
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_project ON alert_rules(project_id) WHERE is_enabled = true;

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'api_key', 'system', 'scheduler')),
    action          VARCHAR(100) NOT NULL,             -- 'database.create', 'branch.delete', 'backup.restore'
    resource_type   VARCHAR(50) NOT NULL,              -- 'database', 'branch', 'endpoint', 'backup'
    resource_id     UUID NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    request_id      UUID,
    changes         JSONB,                             -- {"before": {...}, "after": {...}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id, created_at DESC);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, created_at DESC) WHERE actor_id IS NOT NULL;
```

## VCS Integration

```sql
CREATE TABLE vcs_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    provider        VARCHAR(20) NOT NULL
                    CHECK (provider IN ('github', 'gitlab', 'bitbucket')),
    repo_owner      VARCHAR(255) NOT NULL,
    repo_name       VARCHAR(255) NOT NULL,
    installation_id VARCHAR(255),
    access_token_encrypted TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, provider)
);

CREATE TABLE vcs_branch_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vcs_integration_id UUID NOT NULL REFERENCES vcs_integrations(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    git_branch      VARCHAR(255) NOT NULL,
    pull_request_number INTEGER,
    auto_delete_on_merge BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(vcs_integration_id, git_branch)
);
```

## AI Features

```sql
CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    branch_id       UUID REFERENCES branches(id),
    type            VARCHAR(30) NOT NULL
                    CHECK (type IN ('index', 'schema', 'query_rewrite', 'scaling', 'cost')),
    severity        VARCHAR(10) NOT NULL DEFAULT 'info'
                    CHECK (severity IN ('critical', 'warning', 'info')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    suggested_sql   TEXT,
    estimated_impact JSONB,
    -- Example: {"query_time_reduction_pct": 45, "storage_increase_mb": 120}
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'accepted', 'dismissed', 'applied', 'expired')),
    applied_by      UUID REFERENCES users(id),
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_recs_db_status ON ai_recommendations(database_id, status);

CREATE TABLE query_patterns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    query_fingerprint VARCHAR(64) NOT NULL,            -- normalized query hash
    sample_query    TEXT NOT NULL,
    call_count      BIGINT NOT NULL DEFAULT 0,
    total_time_ms   NUMERIC(20,3) NOT NULL DEFAULT 0,
    mean_time_ms    NUMERIC(20,3) NOT NULL DEFAULT 0,
    max_time_ms     NUMERIC(20,3) NOT NULL DEFAULT 0,
    rows_returned   BIGINT NOT NULL DEFAULT 0,
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(database_id, query_fingerprint)
);

CREATE INDEX idx_query_patterns_db ON query_patterns(database_id, total_time_ms DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Organization | 3 | organizations, users, organization_members |
| Plans & Billing | 3 | plans, billing_accounts, usage_records |
| Project & Database | 4 | projects, regions, databases, database_configurations |
| Branching & Compute | 3 | branches, compute_endpoints, connection_pools |
| Access Control | 2 | database_roles, api_keys |
| Backups & Recovery | 2 | backups, restore_operations |
| Schema Migrations | 2 | migration_histories, deploy_requests |
| Observability | 3 | metric_definitions, alert_rules, audit_log |
| VCS Integration | 2 | vcs_integrations, vcs_branch_links |
| AI Features | 2 | ai_recommendations, query_patterns |
| **Total** | **26** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, safe for multi-region deployment, and prevents enumeration attacks on the API.

2. **Hierarchical resource ownership via foreign keys** — organization > project > database > branch > endpoint chain is enforced at the database level, preventing orphaned resources.

3. **Soft deletes with `deleted_at`** — databases, branches, and other costly resources use soft deletes so recovery is possible within a retention window.

4. **CHECK constraints for state machines** — resource lifecycle states (provisioning, available, scaling, etc.) are constrained at the database level rather than only in application code.

5. **Dedicated `audit_log` table with JSONB changes** — captures before/after state for compliance (SOC 2, GDPR) while keeping the audit schema stable as other tables evolve.

6. **Separate `usage_records` from billing** — raw metering data is decoupled from billing calculations, allowing re-rating and dispute resolution.

7. **`api_keys` store hashed keys only** — the full key is shown once at creation and never stored; lookup uses the prefix, verification uses bcrypt hash.

8. **Partial indexes for common query patterns** — e.g., `WHERE is_default = true` on branches, `WHERE status != 'deleted'` on databases, to keep index sizes small.

9. **VCS integration as a separate concern** — Git provider details are isolated from core database management, allowing the platform to operate without VCS integration for simpler deployments.

10. **AI recommendations as first-class entities** — recommendations have their own lifecycle (pending > accepted > applied) rather than being ephemeral console widgets, enabling tracking of AI value delivered.
