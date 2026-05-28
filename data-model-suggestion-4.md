# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Database-as-a-Service Platform · Created: 2026-05-20

## Philosophy

This model combines a relational core for operational CRUD with a property graph layer for relationship-heavy queries. A DBaaS platform has deeply nested resource hierarchies (organization > project > database > branch > endpoint), cross-cutting relationships (users belong to many organizations, API keys grant access across projects, branches fork from other branches), and dependency chains (endpoints depend on branches, which depend on databases, which depend on regions and compute capacity).

The graph layer uses PostgreSQL's native `pg_graphql` extension or Apache AGE (both provide property graph queries on standard PostgreSQL) to model these relationships as nodes and edges. This enables queries like "find all resources affected if region us-east-1 goes down," "trace the branch lineage from this endpoint back to the production branch," or "show all resources accessible by this API key" — queries that require recursive CTEs or multiple joins in a purely relational model but are single traversals in a graph.

AWS Neptune is used internally by Amazon for dependency mapping in their cloud infrastructure. Google's Spanner uses graph-based metadata for resource dependency tracking. The SQL:2023 standard added SQL/PGQ (Property Graph Queries) in Part 16, formalizing graph queries as part of the SQL standard — PostgreSQL AGE implements this.

**Best for:** Platforms that need to answer complex dependency and lineage questions, perform impact analysis (what breaks if X fails?), visualize resource topologies, or detect circular dependencies and access control conflicts.

**Trade-offs:**
- Pro: Relationship queries (lineage, impact analysis, dependency chains) are orders of magnitude faster than recursive CTEs
- Pro: Visual topology maps come naturally from the graph structure
- Pro: Access control traversal ("can user X reach resource Y?") is a single graph query
- Pro: Branch lineage and fork history are first-class graph traversals
- Pro: SQL/PGQ (SQL:2023 Part 16) provides a standards-based query language
- Con: Requires pg_graphql or Apache AGE extension (not available on all managed PostgreSQL providers)
- Con: Dual write path (relational + graph) adds complexity and potential consistency issues
- Con: Team must learn Cypher or SQL/PGQ in addition to standard SQL
- Con: Graph queries are less familiar to most developers and harder to debug
- Con: Write performance is slightly lower due to maintaining both relational tables and graph edges

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 9075-16:2023 (SQL/PGQ) | Graph queries use the SQL:2023 Property Graph Query standard via Apache AGE |
| ISO/IEC 9075:2023 (SQL) | Core relational tables use standard SQL DDL and DML |
| OpenCypher | Cypher query language used by Apache AGE for graph traversals |
| OAuth 2.0 (RFC 6749) | Access control graph models token scopes as edges between API keys and resources |
| OpenTelemetry | Dependency graph enables automated service map generation for observability |
| ISO 3166-1/2 | Region nodes in the graph carry ISO 3166 country codes as properties |

---

## Relational Core (Operational CRUD)

```sql
-- Standard relational tables for operational data.
-- These serve the REST API for CRUD operations.

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_name       VARCHAR(100) NOT NULL DEFAULT 'free',
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'deleted')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    default_region  VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE TABLE databases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    engine          VARCHAR(30) NOT NULL DEFAULT 'postgresql',
    engine_version  VARCHAR(20) NOT NULL,
    region          VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'provisioning',
    storage_size_mb BIGINT NOT NULL DEFAULT 0,
    config          JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    database_id     UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    parent_branch_id UUID REFERENCES branches(id),
    parent_lsn      VARCHAR(30),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compute_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    host            VARCHAR(255) NOT NULL UNIQUE,
    port            INTEGER NOT NULL DEFAULT 5432,
    type            VARCHAR(20) NOT NULL DEFAULT 'read_write',
    min_cu          NUMERIC(5,2) NOT NULL DEFAULT 0.25,
    max_cu          NUMERIC(5,2) NOT NULL DEFAULT 4.0,
    current_cu      NUMERIC(5,2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'creating',
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(10) NOT NULL,
    key_hash        TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE backups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    type            VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    size_bytes      BIGINT,
    storage_path    TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

## Graph Layer (Apache AGE)

```sql
-- Initialize Apache AGE extension and create the resource graph
CREATE EXTENSION IF NOT EXISTS age;
SET search_path = ag_catalog, "$user", public;

-- Create the graph namespace
SELECT create_graph('dbaas_graph');
```

### Graph Node Types

```sql
-- Nodes mirror the relational entities but carry only graph-relevant properties.
-- The UUID links back to the relational table for full data.

-- Organization node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Organization {
        id: '110e8400-e29b-41d4-a716-446655440000',
        name: 'Acme Corp',
        slug: 'acme-corp',
        plan: 'pro',
        status: 'active'
    })
$$) AS (v agtype);

-- User node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:User {
        id: '220e8400-e29b-41d4-a716-446655440000',
        email: 'alice@acme.com',
        display_name: 'Alice Chen'
    })
$$) AS (v agtype);

-- Project node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Project {
        id: '330e8400-e29b-41d4-a716-446655440000',
        name: 'Backend API',
        slug: 'backend-api',
        region: 'us-east-1'
    })
$$) AS (v agtype);

-- Database node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Database {
        id: '440e8400-e29b-41d4-a716-446655440000',
        name: 'production-db',
        engine: 'postgresql',
        version: '16',
        region: 'us-east-1',
        status: 'available'
    })
$$) AS (v agtype);

-- Branch node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Branch {
        id: '550e8400-e29b-41d4-a716-446655440000',
        name: 'main',
        is_default: true,
        is_protected: true,
        status: 'ready'
    })
$$) AS (v agtype);

-- Endpoint node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Endpoint {
        id: '660e8400-e29b-41d4-a716-446655440000',
        host: 'ep-cool-name-123.us-east-1.dbaas.example.com',
        type: 'read_write',
        current_cu: 2.0,
        status: 'active'
    })
$$) AS (v agtype);

-- Region node
SELECT * FROM cypher('dbaas_graph', $$
    CREATE (:Region {
        id: 'us-east-1',
        cloud: 'aws',
        country: 'US',
        display_name: 'US East (N. Virginia)'
    })
$$) AS (v agtype);
```

### Graph Edge Types

```sql
-- Ownership/containment edges
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (o:Organization {id: '110e8400-e29b-41d4-a716-446655440000'}),
          (p:Project {id: '330e8400-e29b-41d4-a716-446655440000'})
    CREATE (o)-[:OWNS_PROJECT {since: '2026-01-15'}]->(p)
$$) AS (e agtype);

SELECT * FROM cypher('dbaas_graph', $$
    MATCH (p:Project {id: '330e8400-e29b-41d4-a716-446655440000'}),
          (d:Database {id: '440e8400-e29b-41d4-a716-446655440000'})
    CREATE (p)-[:CONTAINS_DATABASE]->(d)
$$) AS (e agtype);

SELECT * FROM cypher('dbaas_graph', $$
    MATCH (d:Database {id: '440e8400-e29b-41d4-a716-446655440000'}),
          (b:Branch {id: '550e8400-e29b-41d4-a716-446655440000'})
    CREATE (d)-[:HAS_BRANCH]->(b)
$$) AS (e agtype);

SELECT * FROM cypher('dbaas_graph', $$
    MATCH (b:Branch {id: '550e8400-e29b-41d4-a716-446655440000'}),
          (e:Endpoint {id: '660e8400-e29b-41d4-a716-446655440000'})
    CREATE (b)-[:SERVES_ENDPOINT]->(e)
$$) AS (e agtype);

-- Branch lineage edge (fork relationship)
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (parent:Branch {id: '550e8400-e29b-41d4-a716-446655440000'}),
          (child:Branch {id: '770e8400-e29b-41d4-a716-446655440000'})
    CREATE (child)-[:FORKED_FROM {
        lsn: '0/1A2B3C4D',
        timestamp: '2026-05-20T10:30:00Z',
        git_branch: 'feature/user-profiles',
        pr_number: 142
    }]->(parent)
$$) AS (e agtype);

-- User membership edge with role
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (u:User {id: '220e8400-e29b-41d4-a716-446655440000'}),
          (o:Organization {id: '110e8400-e29b-41d4-a716-446655440000'})
    CREATE (u)-[:MEMBER_OF {role: 'admin', since: '2026-01-15'}]->(o)
$$) AS (e agtype);

-- API key access edge
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (k:ApiKey {id: '880e8400-e29b-41d4-a716-446655440000'}),
          (p:Project {id: '330e8400-e29b-41d4-a716-446655440000'})
    CREATE (k)-[:GRANTS_ACCESS {scopes: ['databases:read', 'branches:write']}]->(p)
$$) AS (e agtype);

-- Region deployment edge
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (d:Database {id: '440e8400-e29b-41d4-a716-446655440000'}),
          (r:Region {id: 'us-east-1'})
    CREATE (d)-[:DEPLOYED_IN]->(r)
$$) AS (e agtype);
```

---

## Graph Query Examples

### Impact Analysis: What breaks if a region goes down?

```sql
-- Find all resources deployed in us-east-1 and their dependent chain
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (r:Region {id: 'us-east-1'})<-[:DEPLOYED_IN]-(d:Database)
          <-[:HAS_BRANCH]-(b:Branch)
          <-[:SERVES_ENDPOINT]-(e:Endpoint)
    MATCH (d)<-[:CONTAINS_DATABASE]-(p:Project)
          <-[:OWNS_PROJECT]-(o:Organization)
    RETURN o.name AS org, p.name AS project, d.name AS database,
           b.name AS branch, e.host AS endpoint, e.status AS status
$$) AS (org agtype, project agtype, database agtype,
        branch agtype, endpoint agtype, status agtype);
```

### Branch Lineage: Trace a branch back to its root

```sql
-- Recursive traversal of branch fork history
SELECT * FROM cypher('dbaas_graph', $$
    MATCH path = (leaf:Branch {id: '990e8400-e29b-41d4-a716-446655440000'})
                 -[:FORKED_FROM*]->(root:Branch)
    WHERE NOT EXISTS((root)-[:FORKED_FROM]->())
    RETURN [n IN nodes(path) | n.name] AS lineage,
           length(path) AS depth
$$) AS (lineage agtype, depth agtype);
```

### Access Control: Can user X reach resource Y?

```sql
-- Check if a user has a path to a specific database through membership and ownership
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (u:User {id: '220e8400-e29b-41d4-a716-446655440000'})
          -[:MEMBER_OF]->(o:Organization)
          -[:OWNS_PROJECT]->(p:Project)
          -[:CONTAINS_DATABASE]->(d:Database {id: '440e8400-e29b-41d4-a716-446655440000'})
    RETURN u.email AS user, o.name AS org,
           p.name AS project, d.name AS database
$$) AS (user_email agtype, org agtype, project agtype, database agtype);
```

### Resource Topology: Full dependency tree for a project

```sql
-- Get the complete resource tree under a project
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (p:Project {id: '330e8400-e29b-41d4-a716-446655440000'})
          -[:CONTAINS_DATABASE]->(d:Database)
    OPTIONAL MATCH (d)-[:HAS_BRANCH]->(b:Branch)
    OPTIONAL MATCH (b)-[:SERVES_ENDPOINT]->(e:Endpoint)
    OPTIONAL MATCH (b)<-[:FORKED_FROM]-(child:Branch)
    RETURN d.name AS database, d.status AS db_status,
           collect(DISTINCT {name: b.name, status: b.status}) AS branches,
           collect(DISTINCT {host: e.host, status: e.status}) AS endpoints,
           count(DISTINCT child) AS child_branches
$$) AS (database agtype, db_status agtype,
        branches agtype, endpoints agtype, child_branches agtype);
```

### Multi-Tenant Isolation Check: Find cross-org resource sharing (shouldn't exist)

```sql
-- Detect any resources accessible from multiple organizations (security check)
SELECT * FROM cypher('dbaas_graph', $$
    MATCH (o1:Organization)-[:OWNS_PROJECT]->(:Project)-[:CONTAINS_DATABASE]->(d:Database)
          <-[:CONTAINS_DATABASE]-(:Project)<-[:OWNS_PROJECT]-(o2:Organization)
    WHERE o1.id <> o2.id
    RETURN o1.name AS org1, o2.name AS org2, d.name AS shared_database
$$) AS (org1 agtype, org2 agtype, shared_database agtype);
```

---

## Synchronization Between Relational and Graph

```sql
-- Trigger function to keep graph in sync with relational writes
-- (simplified example for databases table)

CREATE OR REPLACE FUNCTION sync_database_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- Create node in graph
        PERFORM * FROM cypher('dbaas_graph', format($$
            CREATE (:Database {
                id: %L, name: %L, engine: %L,
                version: %L, region: %L, status: %L
            })
        $$, NEW.id, NEW.name, NEW.engine,
            NEW.engine_version, NEW.region, NEW.status))
        AS (v agtype);

        -- Create containment edge
        PERFORM * FROM cypher('dbaas_graph', format($$
            MATCH (p:Project {id: %L}), (d:Database {id: %L})
            CREATE (p)-[:CONTAINS_DATABASE]->(d)
        $$, NEW.project_id, NEW.id))
        AS (e agtype);

        -- Create region edge
        PERFORM * FROM cypher('dbaas_graph', format($$
            MATCH (d:Database {id: %L}), (r:Region {id: %L})
            CREATE (d)-[:DEPLOYED_IN]->(r)
        $$, NEW.id, NEW.region))
        AS (e agtype);

    ELSIF TG_OP = 'UPDATE' THEN
        PERFORM * FROM cypher('dbaas_graph', format($$
            MATCH (d:Database {id: %L})
            SET d.status = %L, d.name = %L
        $$, NEW.id, NEW.status, NEW.name))
        AS (v agtype);

    ELSIF TG_OP = 'DELETE' THEN
        PERFORM * FROM cypher('dbaas_graph', format($$
            MATCH (d:Database {id: %L})
            DETACH DELETE d
        $$, OLD.id))
        AS (v agtype);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_database_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON databases
    FOR EACH ROW EXECUTE FUNCTION sync_database_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational Core | 9 | organizations, users, projects, databases, branches, compute_endpoints, api_keys, backups, audit_log |
| Graph Layer | 1 | Apache AGE graph `dbaas_graph` (contains all node labels and edge types) |
| Sync Infrastructure | 0 | Triggers on relational tables, no additional tables |
| **Total** | **10** | Plus the graph which stores nodes/edges internally via AGE |

### Graph Node Labels

| Label | Properties | Relational Source |
|-------|-----------|-------------------|
| Organization | id, name, slug, plan, status | organizations |
| User | id, email, display_name | users |
| Project | id, name, slug, region | projects |
| Database | id, name, engine, version, region, status | databases |
| Branch | id, name, is_default, is_protected, status | branches |
| Endpoint | id, host, type, current_cu, status | compute_endpoints |
| Region | id, cloud, country, display_name | (reference data) |
| ApiKey | id, name, scopes | api_keys |

### Graph Edge Types

| Edge | From | To | Properties |
|------|------|----|-----------|
| MEMBER_OF | User | Organization | role, since |
| OWNS_PROJECT | Organization | Project | since |
| CONTAINS_DATABASE | Project | Database | |
| HAS_BRANCH | Database | Branch | |
| SERVES_ENDPOINT | Branch | Endpoint | |
| FORKED_FROM | Branch | Branch | lsn, timestamp, git_branch, pr_number |
| DEPLOYED_IN | Database | Region | |
| GRANTS_ACCESS | ApiKey | Project | scopes |
| CREATED_BY | Resource | User | at |

---

## Key Design Decisions

1. **Dual storage: relational for CRUD, graph for traversal** — operational API endpoints read/write relational tables for performance and simplicity. Graph queries are used for topology visualization, impact analysis, and access control checks.

2. **Apache AGE over external graph database** — AGE runs as a PostgreSQL extension, eliminating the operational overhead of a separate graph database (Neo4j). The graph is queryable from the same connection pool and transactional context.

3. **SQL/PGQ alignment** — Cypher queries used by AGE align with the SQL:2023 Part 16 (SQL/PGQ) standard, providing a migration path to native SQL graph queries as PostgreSQL adds built-in support.

4. **Trigger-based synchronization** — relational writes automatically propagate to the graph via triggers, keeping both representations consistent without application-level dual-write logic.

5. **Graph nodes carry minimal properties** — nodes store only graph-relevant properties (id, name, status). Full entity data is retrieved by joining back to relational tables using the UUID.

6. **Branch lineage as first-class edges** — the `FORKED_FROM` edge type with LSN and timestamp properties makes branch genealogy a single graph traversal, which would require recursive CTEs in pure SQL.

7. **Access control as graph traversal** — "can user X reach resource Y?" is answered by a path-finding query rather than joining through membership, project ownership, and database containment tables.

8. **Region as a graph node** — modeling regions as nodes enables impact analysis queries ("what's deployed in us-east-1?") as simple neighbor lookups.

9. **Fewer relational tables (10)** — the graph absorbs relationship complexity that would require junction tables and recursive structures in a purely relational model.

10. **Security isolation verification** — graph queries can detect cross-organization resource sharing, circular dependencies, or orphaned resources that would be expensive to find with relational queries alone.
