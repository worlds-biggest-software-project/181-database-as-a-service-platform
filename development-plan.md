# Database-as-a-Service Platform — Phased Development Plan

> Project: 181-database-as-a-service-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js) | API-heavy platform with management console; TypeScript provides type safety for the complex resource hierarchy (org > project > database > branch > endpoint), strong OpenAPI tooling ecosystem, and first-class async/await for orchestrating provisioning workflows and long-running operations |
| API framework | Fastify + @fastify/swagger | Fastify delivers 2-3x throughput over Express with built-in schema validation via JSON Schema; @fastify/swagger auto-generates OpenAPI 3.1 spec from route schemas, meeting the standards requirement without manual spec maintenance |
| Database (control plane) | PostgreSQL 16 | The control plane stores the resource hierarchy, audit logs, and usage records. Data Model Suggestion 3 (Hybrid Relational + JSONB) is adopted — it provides relational integrity for the core resource graph while using JSONB for engine-specific configs, VCS integration details, and rapidly-evolving settings. PostgreSQL's JSONB indexing (GIN), partitioning, and CHECK constraints cover all requirements with 14 tables instead of 26 |
| Database (managed engines) | PostgreSQL 16 + MySQL 8.0 | The platform provisions and manages PostgreSQL as the primary engine (MVP) with MySQL as the second engine (v1.1). The provisioning layer abstracts engine-specific operations behind a common interface |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead; generates migration files; supports JSONB columns natively; schema defined in TypeScript aligning with the hybrid relational+JSONB data model |
| Task queue | BullMQ (Redis-backed) | Provisioning, backup, scaling, and AI recommendation generation are async operations. BullMQ provides job scheduling, retries, progress tracking, and rate limiting. Redis also serves as the cache layer for connection pool metadata and session state |
| Frontend | Next.js 15 (App Router) | Server components reduce client bundle for the management console; API routes proxy to the Fastify backend; Tailwind CSS + shadcn/ui for the dashboard UI. The console needs real-time updates (endpoint status, scaling events) served via Server-Sent Events |
| CLI | Commander.js | Lightweight CLI framework for the `dbaas` CLI tool; supports subcommands (`dbaas database create`, `dbaas branch list`), config file management, and output formatting (table, JSON, YAML) |
| Authentication | Lucia Auth + Oslo | Lucia provides session-based auth for the console; Oslo handles OAuth 2.0 Client Credentials for API keys and JWT issuance for database connection tokens. Supports GitHub, Google, and SAML 2.0 SSO |
| Containerisation | Docker + Docker Compose | Multi-container setup: API server, worker (BullMQ processor), PostgreSQL (control plane), Redis, and the Next.js console. Production deployment targets Kubernetes with Helm charts |
| Compute orchestration | Kubernetes API (client-node) | Database compute endpoints are Kubernetes pods. The provisioning service uses the Kubernetes API to create, scale, and destroy compute pods. Scale-to-zero is implemented via KEDA (Kubernetes Event-Driven Autoscaling) watching connection count metrics |
| Observability | OpenTelemetry SDK + Prometheus | OTel SDK instruments the API and worker with traces and metrics. Prometheus scrapes metrics from the control plane and managed database endpoints. Structured JSON logging via Pino (Fastify's default logger) |
| Testing framework | Vitest | Fast, TypeScript-native test runner with built-in mocking, code coverage, and concurrent test execution. Compatible with the Fastify testing utilities |
| Code quality | ESLint + Prettier + tsc --noEmit | ESLint with typescript-eslint for linting, Prettier for formatting, TypeScript compiler for type checking. Husky + lint-staged for pre-commit hooks |
| Package manager | pnpm | Workspace-aware package manager for the monorepo (API, console, CLI, shared types). Strict dependency isolation prevents phantom dependencies |
| MCP server | @modelcontextprotocol/sdk | First-party MCP server endpoint exposing database schema introspection, query execution, and branch management tools for AI coding assistants per the MCP 2025-11-25 specification |
| Backup storage | S3-compatible (MinIO for dev) | Backups stored in S3-compatible object storage. MinIO runs locally in Docker Compose for development; production uses AWS S3, GCS, or any S3-compatible store |
| Encryption | Node.js crypto + AWS KMS (production) | AES-256-GCM for encryption at rest; TLS 1.3 enforced for all connections per RFC 8446. API key hashing uses bcrypt. Production key management via AWS KMS or HashiCorp Vault |

### Project Structure

```
dbaas-platform/
├── pnpm-workspace.yaml
├── package.json
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile.api
├── Dockerfile.console
├── .env.example
├── turbo.json                          # Turborepo for monorepo builds
│
├── packages/
│   ├── shared/                         # Shared types, constants, validators
│   │   ├── src/
│   │   │   ├── types/
│   │   │   │   ├── organization.ts
│   │   │   │   ├── project.ts
│   │   │   │   ├── database.ts
│   │   │   │   ├── branch.ts
│   │   │   │   ├── endpoint.ts
│   │   │   │   ├── backup.ts
│   │   │   │   ├── api-key.ts
│   │   │   │   ├── operation.ts
│   │   │   │   └── index.ts
│   │   │   ├── schemas/                # JSON Schema definitions for JSONB validation
│   │   │   │   ├── engine-config.schema.ts
│   │   │   │   ├── security-config.schema.ts
│   │   │   │   ├── compute-config.schema.ts
│   │   │   │   └── vcs-link.schema.ts
│   │   │   ├── constants/
│   │   │   │   ├── engines.ts
│   │   │   │   ├── regions.ts
│   │   │   │   └── limits.ts
│   │   │   └── validators/
│   │   │       └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── db/                             # Drizzle schema + migrations
│       ├── src/
│       │   ├── schema/
│       │   │   ├── organizations.ts
│       │   │   ├── users.ts
│       │   │   ├── projects.ts
│       │   │   ├── databases.ts
│       │   │   ├── branches.ts
│       │   │   ├── compute-endpoints.ts
│       │   │   ├── api-keys.ts
│       │   │   ├── database-roles.ts
│       │   │   ├── backups.ts
│       │   │   ├── operations.ts
│       │   │   ├── audit-log.ts
│       │   │   ├── usage-records.ts
│       │   │   ├── ai-recommendations.ts
│       │   │   └── index.ts
│       │   ├── client.ts
│       │   └── seed.ts
│       ├── drizzle/                    # Generated migration files
│       ├── drizzle.config.ts
│       ├── package.json
│       └── tsconfig.json
│
├── apps/
│   ├── api/                            # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── config.ts
│   │   │   ├── routes/
│   │   │   │   ├── organizations/
│   │   │   │   ├── projects/
│   │   │   │   ├── databases/
│   │   │   │   ├── branches/
│   │   │   │   ├── endpoints/
│   │   │   │   ├── backups/
│   │   │   │   ├── api-keys/
│   │   │   │   ├── operations/
│   │   │   │   ├── ai/
│   │   │   │   └── health.ts
│   │   │   ├── services/
│   │   │   │   ├── provisioning.service.ts
│   │   │   │   ├── branching.service.ts
│   │   │   │   ├── backup.service.ts
│   │   │   │   ├── scaling.service.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── billing.service.ts
│   │   │   │   ├── ai-recommendations.service.ts
│   │   │   │   └── mcp.service.ts
│   │   │   ├── workers/
│   │   │   │   ├── provisioning.worker.ts
│   │   │   │   ├── backup.worker.ts
│   │   │   │   ├── scaling.worker.ts
│   │   │   │   ├── ai-analysis.worker.ts
│   │   │   │   └── cleanup.worker.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── rbac.ts
│   │   │   │   ├── rate-limit.ts
│   │   │   │   ├── audit.ts
│   │   │   │   └── error-handler.ts
│   │   │   ├── plugins/
│   │   │   │   ├── database.ts
│   │   │   │   ├── redis.ts
│   │   │   │   ├── queue.ts
│   │   │   │   └── kubernetes.ts
│   │   │   └── lib/
│   │   │       ├── encryption.ts
│   │   │       ├── slug.ts
│   │   │       └── pagination.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── console/                        # Next.js management console
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── (auth)/
│   │   │   │   ├── (dashboard)/
│   │   │   │   │   ├── organizations/
│   │   │   │   │   ├── projects/
│   │   │   │   │   ├── databases/
│   │   │   │   │   ├── branches/
│   │   │   │   │   ├── settings/
│   │   │   │   │   └── billing/
│   │   │   │   └── api/
│   │   │   ├── components/
│   │   │   │   ├── ui/                 # shadcn/ui components
│   │   │   │   ├── database/
│   │   │   │   ├── branch/
│   │   │   │   ├── endpoint/
│   │   │   │   └── layout/
│   │   │   ├── lib/
│   │   │   │   ├── api-client.ts
│   │   │   │   └── hooks/
│   │   │   └── styles/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── cli/                            # dbaas CLI tool
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── commands/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── database.ts
│   │   │   │   ├── branch.ts
│   │   │   │   ├── endpoint.ts
│   │   │   │   ├── backup.ts
│   │   │   │   └── config.ts
│   │   │   ├── config.ts
│   │   │   └── output.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── mcp-server/                     # MCP server for AI assistants
│       ├── src/
│       │   ├── index.ts
│       │   ├── tools/
│       │   │   ├── schema-introspect.ts
│       │   │   ├── query-execute.ts
│       │   │   ├── branch-manage.ts
│       │   │   └── recommend.ts
│       │   └── auth.ts
│       ├── package.json
│       └── tsconfig.json
│
└── tests/
    ├── fixtures/
    │   ├── seed-data.sql
    │   ├── sample-engine-configs/
    │   └── sample-backups/
    ├── helpers/
    │   ├── test-server.ts
    │   ├── test-db.ts
    │   └── factories.ts
    └── e2e/
        ├── database-lifecycle.test.ts
        ├── branching.test.ts
        ├── backup-restore.test.ts
        └── api-key-auth.test.ts
```

---

## Phase 1: Foundation — Project Scaffold, Database Schema, and Configuration

### Purpose

Establish the monorepo structure, install dependencies, configure build tooling, define the control plane database schema using Drizzle ORM (based on Data Model Suggestion 3: Hybrid Relational + JSONB), and create the foundational configuration system. After this phase, the project builds, lints, type-checks, and connects to a running PostgreSQL instance with the schema applied.

### Tasks

#### 1.1 — Monorepo Scaffold and Build Tooling

**What**: Initialize the pnpm workspace with all packages and apps, configure Turborepo for build orchestration, and set up code quality tooling.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

Root `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "typecheck": {}
  }
}
```

Shared ESLint config (`.eslintrc.cjs`):
```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  rules: {
    "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
    "@typescript-eslint/explicit-function-return-type": "warn"
  }
};
```

Shared `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors across all workspaces`
- `Unit: pnpm turbo build compiles all packages and apps to dist/`
- `Unit: pnpm turbo lint passes with zero warnings on initial scaffold`
- `Unit: pnpm turbo typecheck passes with zero errors`
- `Unit: .env.example contains all required environment variables with documented defaults`

---

#### 1.2 — Shared Types and Constants

**What**: Define TypeScript types for all core domain entities, matching the Hybrid Relational + JSONB data model.

**Design**:

```typescript
// packages/shared/src/types/database.ts

export type DatabaseEngine = "postgresql" | "mysql";

export type DatabaseStatus =
  | "provisioning"
  | "available"
  | "scaling"
  | "suspended"
  | "deleting"
  | "deleted"
  | "error";

export type BranchStatus = "creating" | "ready" | "deleting" | "deleted" | "error";

export type EndpointStatus = "creating" | "active" | "idle" | "suspended" | "error";

export type EndpointType = "read_write" | "read_only";

export type OperationStatus = "pending" | "running" | "completed" | "failed" | "cancelled";

export type OrgRole = "owner" | "admin" | "member" | "viewer";

export interface EngineConfig {
  // PostgreSQL
  max_connections?: number;
  shared_buffers?: string;
  work_mem?: string;
  maintenance_work_mem?: string;
  effective_cache_size?: string;
  wal_level?: string;
  max_wal_senders?: number;
  extensions_enabled?: string[];
  pg_bouncer?: {
    pool_mode: "session" | "transaction" | "statement";
    max_client_connections: number;
    default_pool_size: number;
  };
  // MySQL
  innodb_buffer_pool_size?: string;
  innodb_flush_log_at_trx_commit?: number;
  binlog_format?: string;
  character_set_server?: string;
}

export interface SecurityConfig {
  encryption_at_rest: boolean;
  encryption_algorithm: string;
  kms_key_id?: string;
  min_tls_version: string;
  require_ssl: boolean;
  ip_allowlist?: string[];
}

export interface ComputeConfig {
  min_cu: number;
  max_cu: number;
  current_cu: number;
  autoscaling: boolean;
  scale_to_zero: boolean;
  idle_timeout_seconds: number;
  suspend_timeout_seconds?: number;
  provisioner?: string;
  instance_type?: string;
  availability_zone?: string;
}

export interface ConnectionPoolingConfig {
  enabled: boolean;
  pool_mode: "session" | "transaction" | "statement";
  pooler_host: string;
  pooler_port: number;
  max_client_connections: number;
  default_pool_size: number;
}

export interface BranchPoint {
  parent_lsn: string;
  parent_timestamp: string;
  cow_storage_path?: string;
  data_size_at_fork_mb?: number;
}

export interface VcsLink {
  git_branch: string;
  pull_request_number?: number;
  pull_request_url?: string;
  auto_delete_on_merge: boolean;
}

export interface ProjectSettings {
  default_region?: string;
  default_engine?: DatabaseEngine;
  observability?: {
    prometheus_endpoint?: string;
    otel_endpoint?: string;
    log_level?: string;
  };
  vcs_integration?: {
    provider: "github" | "gitlab" | "bitbucket";
    repo_owner: string;
    repo_name: string;
    installation_id?: string;
    auto_branch: boolean;
    auto_delete_on_merge: boolean;
  };
  ip_allowlist?: string[];
}

export interface OrgSettings {
  billing_email?: string;
  default_region?: string;
  sso_config?: {
    provider: string;
    domain: string;
    client_id: string;
    protocol: "saml" | "oidc";
  };
  compliance?: {
    data_residency?: string[];
    require_encryption: boolean;
    audit_log_retention_days: number;
  };
  feature_flags?: Record<string, boolean>;
}

export interface OrgPlan {
  name: string;
  max_projects: number | null;
  max_databases: number | null;
  max_branches_per_db: number | null;
  max_compute_cu: number | null;
  backup_retention_days: number;
  pitr_enabled: boolean;
  price_monthly_cents: number;
}
```

```typescript
// packages/shared/src/constants/engines.ts

export const SUPPORTED_ENGINES = {
  postgresql: {
    versions: ["16", "15", "14"],
    defaultVersion: "16",
    defaultPort: 5432,
    wireProtocol: "postgresql",
  },
  mysql: {
    versions: ["8.0", "8.4"],
    defaultVersion: "8.0",
    defaultPort: 3306,
    wireProtocol: "mysql",
  },
} as const;
```

```typescript
// packages/shared/src/constants/regions.ts

export interface Region {
  id: string;
  cloudProvider: "aws" | "gcp" | "azure";
  displayName: string;
  countryCode: string; // ISO 3166-1 alpha-2
}

export const REGIONS: Region[] = [
  { id: "us-east-1", cloudProvider: "aws", displayName: "US East (N. Virginia)", countryCode: "US" },
  { id: "us-west-2", cloudProvider: "aws", displayName: "US West (Oregon)", countryCode: "US" },
  { id: "eu-west-1", cloudProvider: "aws", displayName: "EU West (Ireland)", countryCode: "IE" },
  { id: "eu-central-1", cloudProvider: "aws", displayName: "EU Central (Frankfurt)", countryCode: "DE" },
  { id: "ap-southeast-1", cloudProvider: "aws", displayName: "Asia Pacific (Singapore)", countryCode: "SG" },
];
```

```typescript
// packages/shared/src/constants/limits.ts

export const PLAN_LIMITS = {
  free: {
    max_projects: 1,
    max_databases: 1,
    max_branches_per_db: 5,
    max_compute_cu: 0.25,
    backup_retention_days: 1,
    pitr_enabled: false,
    price_monthly_cents: 0,
  },
  pro: {
    max_projects: 25,
    max_databases: 100,
    max_branches_per_db: 50,
    max_compute_cu: 8.0,
    backup_retention_days: 14,
    pitr_enabled: true,
    price_monthly_cents: 2500,
  },
  team: {
    max_projects: null, // unlimited
    max_databases: null,
    max_branches_per_db: null,
    max_compute_cu: 16.0,
    backup_retention_days: 30,
    pitr_enabled: true,
    price_monthly_cents: 10000,
  },
} as const;
```

**Testing**:
- `Unit: all type exports are importable from @dbaas/shared`
- `Unit: SUPPORTED_ENGINES.postgresql.defaultVersion equals "16"`
- `Unit: REGIONS array contains at least 5 regions with valid ISO 3166-1 country codes`
- `Unit: PLAN_LIMITS.free.max_compute_cu equals 0.25`
- `Unit: DatabaseStatus union type accepts all 7 valid statuses`

---

#### 1.3 — Drizzle Schema Definition (Control Plane Database)

**What**: Define the complete control plane database schema in Drizzle ORM, matching Data Model Suggestion 3 (Hybrid Relational + JSONB), and generate the initial migration.

**Design**:

```typescript
// packages/db/src/schema/organizations.ts

import { pgTable, uuid, varchar, text, jsonb, timestamp, uniqueIndex, index } from "drizzle-orm/pg-core";

export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  settings: jsonb("settings").notNull().default({}),   // OrgSettings
  plan: jsonb("plan").notNull().default({}),            // OrgPlan
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  slugIdx: uniqueIndex("idx_org_slug").on(table.slug),
}));

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  displayName: varchar("display_name", { length: 255 }),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  profile: jsonb("profile").notNull().default({}),
  passwordHash: text("password_hash"),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMembers = pgTable("organization_members", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  role: varchar("role", { length: 30 }).notNull().default("member"),
  permissions: jsonb("permissions").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index("idx_orgmem_org").on(table.organizationId),
  userIdx: index("idx_orgmem_user").on(table.userId),
  orgUserUnique: uniqueIndex("idx_orgmem_org_user").on(table.organizationId, table.userId),
}));
```

```typescript
// packages/db/src/schema/databases.ts

import { pgTable, uuid, varchar, bigint, jsonb, timestamp, index } from "drizzle-orm/pg-core";
import { projects } from "./projects";
import { users } from "./organizations";

export const databases = pgTable("databases", {
  id: uuid("id").primaryKey().defaultRandom(),
  projectId: uuid("project_id").notNull().references(() => projects.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  engine: varchar("engine", { length: 30 }).notNull().default("postgresql"),
  engineVersion: varchar("engine_version", { length: 20 }).notNull(),
  region: varchar("region", { length: 30 }).notNull(),
  cloudProvider: varchar("cloud_provider", { length: 20 }).notNull().default("aws"),
  status: varchar("status", { length: 30 }).notNull().default("provisioning"),
  storageSizeMb: bigint("storage_size_mb", { mode: "number" }).notNull().default(0),
  engineConfig: jsonb("engine_config").notNull().default({}),     // EngineConfig
  security: jsonb("security").notNull().default({}),              // SecurityConfig
  createdBy: uuid("created_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
}, (table) => ({
  projectIdx: index("idx_db_project").on(table.projectId),
  statusIdx: index("idx_db_status").on(table.status),
}));
```

```typescript
// packages/db/src/schema/branches.ts

import { pgTable, uuid, varchar, boolean, jsonb, timestamp, index, uniqueIndex } from "drizzle-orm/pg-core";
import { databases } from "./databases";
import { users } from "./organizations";

export const branches = pgTable("branches", {
  id: uuid("id").primaryKey().defaultRandom(),
  databaseId: uuid("database_id").notNull().references(() => databases.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  parentBranchId: uuid("parent_branch_id").references((): any => branches.id),
  isDefault: boolean("is_default").notNull().default(false),
  isProtected: boolean("is_protected").notNull().default(false),
  status: varchar("status", { length: 20 }).notNull().default("creating"),
  branchPoint: jsonb("branch_point"),             // BranchPoint
  vcsLink: jsonb("vcs_link"),                     // VcsLink
  createdBy: uuid("created_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
}, (table) => ({
  dbIdx: index("idx_branches_db").on(table.databaseId),
  defaultIdx: uniqueIndex("idx_branches_default").on(table.databaseId).where(sql`is_default = true`),
}));
```

```typescript
// packages/db/src/schema/compute-endpoints.ts

import { pgTable, uuid, varchar, integer, jsonb, timestamp, index, uniqueIndex } from "drizzle-orm/pg-core";
import { branches } from "./branches";

export const computeEndpoints = pgTable("compute_endpoints", {
  id: uuid("id").primaryKey().defaultRandom(),
  branchId: uuid("branch_id").notNull().references(() => branches.id, { onDelete: "cascade" }),
  host: varchar("host", { length: 255 }).notNull().unique(),
  port: integer("port").notNull().default(5432),
  type: varchar("type", { length: 20 }).notNull().default("read_write"),
  status: varchar("status", { length: 20 }).notNull().default("creating"),
  compute: jsonb("compute").notNull().default({}),               // ComputeConfig
  connectionPooling: jsonb("connection_pooling"),                 // ConnectionPoolingConfig
  lastActiveAt: timestamp("last_active_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  branchIdx: index("idx_endpoints_branch").on(table.branchId),
  statusIdx: index("idx_endpoints_status").on(table.status),
}));
```

Remaining schema tables follow the same pattern for: `projects`, `database_roles`, `api_keys`, `backups`, `operations`, `audit_log` (partitioned by `created_at`), `usage_records` (partitioned by `period_start`), and `ai_recommendations`.

Database client initialization:

```typescript
// packages/db/src/client.ts

import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

export function createDbClient(connectionString: string) {
  const pool = new Pool({ connectionString, max: 20 });
  return drizzle(pool, { schema });
}

export type DbClient = ReturnType<typeof createDbClient>;
```

**Testing**:
- `Unit: drizzle-kit generate produces a valid SQL migration file`
- `Integration (real DB): drizzle-kit migrate applies all tables to a fresh PostgreSQL database`
- `Integration (real DB): inserting an organization with JSONB settings succeeds and is queryable via @> containment`
- `Integration (real DB): foreign key cascade deletes a project's databases when the project is deleted`
- `Integration (real DB): unique constraint on (organization_id, slug) for projects prevents duplicates`
- `Integration (real DB): inserting a database with invalid status value is rejected by CHECK constraint`

---

#### 1.4 — Configuration System and Docker Compose

**What**: Create the application configuration module, `.env.example`, and `docker-compose.yml` for local development.

**Design**:

```typescript
// apps/api/src/config.ts

import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default("0.0.0.0"),

  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default("redis://localhost:6379"),

  JWT_SECRET: z.string().min(32),
  JWT_EXPIRY: z.string().default("1h"),

  API_KEY_SALT_ROUNDS: z.coerce.number().default(12),

  S3_ENDPOINT: z.string().url().optional(),
  S3_BUCKET: z.string().default("dbaas-backups"),
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),

  KUBERNETES_API_URL: z.string().url().optional(),
  KUBERNETES_NAMESPACE: z.string().default("dbaas-managed"),

  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),

  OTEL_EXPORTER_OTLP_ENDPOINT: z.string().url().optional(),

  CORS_ORIGINS: z.string().default("http://localhost:3000"),
});

export type Config = z.infer<typeof envSchema>;

export function loadConfig(): Config {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error("Invalid environment configuration:", result.error.flatten().fieldErrors);
    process.exit(1);
  }
  return result.data;
}
```

```yaml
# docker-compose.yml

services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: dbaas_control
      POSTGRES_USER: dbaas
      POSTGRES_PASSWORD: dbaas_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dbaas"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s

  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data

volumes:
  pgdata:
  minio-data:
```

**Testing**:
- `Unit: loadConfig() with valid .env returns typed Config object`
- `Unit: loadConfig() with missing DATABASE_URL exits with descriptive error`
- `Unit: loadConfig() with PORT="abc" exits with coercion error`
- `Integration: docker-compose up starts postgres, redis, and minio; all health checks pass`
- `Integration: DATABASE_URL from docker-compose connects successfully from the API app`

---

## Phase 2: Core API — Authentication, Organizations, and Projects

### Purpose

Build the Fastify API server with authentication (API keys and session-based), organization management, and project CRUD. After this phase, users can create accounts, create organizations, invite members, manage projects, and authenticate via API keys or browser sessions. The management API serves OpenAPI 3.1 documentation automatically.

### Tasks

#### 2.1 — Fastify Server Bootstrap with OpenAPI

**What**: Initialize the Fastify server with plugin architecture, OpenAPI 3.1 documentation, error handling, request logging, and health check endpoint.

**Design**:

```typescript
// apps/api/src/server.ts

import Fastify, { FastifyInstance } from "fastify";
import fastifySwagger from "@fastify/swagger";
import fastifySwaggerUi from "@fastify/swagger-ui";
import fastifyCors from "@fastify/cors";
import { loadConfig } from "./config";

export async function buildServer(): Promise<FastifyInstance> {
  const config = loadConfig();

  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === "development"
        ? { target: "pino-pretty" }
        : undefined,
    },
    genReqId: () => crypto.randomUUID(),
  });

  // OpenAPI 3.1 documentation
  await app.register(fastifySwagger, {
    openapi: {
      openapi: "3.1.0",
      info: {
        title: "DBaaS Platform API",
        version: "1.0.0",
        description: "Database-as-a-Service management API",
      },
      servers: [{ url: `http://localhost:${config.PORT}` }],
      components: {
        securitySchemes: {
          apiKey: {
            type: "http",
            scheme: "bearer",
            description: "API key (Bearer dbaas_...)",
          },
          session: {
            type: "apiKey",
            in: "cookie",
            name: "session_id",
          },
        },
      },
    },
  });

  await app.register(fastifySwaggerUi, { routePrefix: "/docs" });
  await app.register(fastifyCors, { origin: config.CORS_ORIGINS.split(","), credentials: true });

  // Health check
  app.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string", enum: ["healthy"] },
            version: { type: "string" },
            timestamp: { type: "string", format: "date-time" },
          },
        },
      },
    },
  }, async () => ({
    status: "healthy" as const,
    version: process.env.npm_package_version ?? "0.0.0",
    timestamp: new Date().toISOString(),
  }));

  return app;
}
```

Error handler middleware:

```typescript
// apps/api/src/middleware/error-handler.ts

import { FastifyError, FastifyReply, FastifyRequest } from "fastify";

export interface ApiError {
  statusCode: number;
  error: string;
  message: string;
  requestId?: string;
}

export class AppError extends Error {
  constructor(
    public statusCode: number,
    public error: string,
    message: string,
  ) {
    super(message);
  }
}

export function errorHandler(
  error: FastifyError,
  request: FastifyRequest,
  reply: FastifyReply,
): void {
  const statusCode = error instanceof AppError ? error.statusCode : (error.statusCode ?? 500);
  const response: ApiError = {
    statusCode,
    error: error instanceof AppError ? error.error : "Internal Server Error",
    message: statusCode >= 500 ? "An unexpected error occurred" : error.message,
    requestId: request.id,
  };

  request.log.error({ err: error, requestId: request.id }, "Request error");
  reply.status(statusCode).send(response);
}
```

**Testing**:
- `Unit: buildServer() returns a Fastify instance that listens on the configured port`
- `Unit: GET /health returns 200 with status "healthy" and ISO 8601 timestamp`
- `Unit: GET /docs returns 200 with Swagger UI HTML`
- `Unit: GET /docs/json returns valid OpenAPI 3.1 JSON document`
- `Unit: unknown route returns 404 with ApiError shape`
- `Unit: error handler returns 500 for unhandled errors with generic message (no stack leak)`
- `Unit: error handler returns AppError statusCode and message for known errors`
- `Unit: every response includes x-request-id header matching the request ID`

---

#### 2.2 — Authentication System (API Keys and Sessions)

**What**: Implement dual authentication: API key authentication (Bearer token) for programmatic access and session-based authentication for the console.

**Design**:

```typescript
// apps/api/src/services/auth.service.ts

import bcrypt from "bcrypt";
import crypto from "node:crypto";
import { DbClient } from "@dbaas/db";

export interface ApiKeyPayload {
  keyId: string;
  organizationId: string;
  userId: string | null;
  scopes: string[];
}

export interface SessionPayload {
  userId: string;
  email: string;
  organizationId: string;
  role: OrgRole;
}

export class AuthService {
  constructor(private db: DbClient, private jwtSecret: string) {}

  // API Key generation: dbaas_<32 random bytes as hex>
  async createApiKey(params: {
    organizationId: string;
    userId?: string;
    name: string;
    scopes: string[];
    expiresAt?: Date;
  }): Promise<{ key: string; keyId: string }> {
    const rawKey = `dbaas_${crypto.randomBytes(32).toString("hex")}`;
    const keyPrefix = rawKey.slice(0, 12);
    const keyHash = await bcrypt.hash(rawKey, 12);
    // Insert into api_keys table
    // Return { key: rawKey, keyId }  (rawKey shown once, never stored)
  }

  async validateApiKey(bearerToken: string): Promise<ApiKeyPayload | null> {
    const prefix = bearerToken.slice(0, 12);
    // Look up by prefix, verify with bcrypt
    // Check expiry, revocation
    // Update last_used_at
  }

  async createSession(userId: string, organizationId: string): Promise<string> {
    // Create session token, store in Redis with TTL
  }

  async validateSession(sessionId: string): Promise<SessionPayload | null> {
    // Look up in Redis, return payload or null
  }
}
```

Auth middleware:

```typescript
// apps/api/src/middleware/auth.ts

import { FastifyRequest, FastifyReply } from "fastify";

declare module "fastify" {
  interface FastifyRequest {
    auth: {
      type: "api_key" | "session";
      userId: string | null;
      organizationId: string;
      scopes: string[];
      role?: OrgRole;
    };
  }
}

export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply,
): Promise<void> {
  const authHeader = request.headers.authorization;
  const sessionCookie = request.cookies?.session_id;

  if (authHeader?.startsWith("Bearer dbaas_")) {
    const payload = await request.server.authService.validateApiKey(authHeader.slice(7));
    if (!payload) { reply.status(401).send({ error: "Invalid API key" }); return; }
    request.auth = { type: "api_key", ...payload };
  } else if (sessionCookie) {
    const payload = await request.server.authService.validateSession(sessionCookie);
    if (!payload) { reply.status(401).send({ error: "Invalid session" }); return; }
    request.auth = { type: "session", ...payload, scopes: ["*"] };
  } else {
    reply.status(401).send({ error: "Authentication required" });
  }
}
```

RBAC middleware:

```typescript
// apps/api/src/middleware/rbac.ts

export function requireScope(...requiredScopes: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    if (request.auth.scopes.includes("*")) return; // session auth has full access
    const hasScope = requiredScopes.some(
      (scope) => request.auth.scopes.includes(scope) || request.auth.scopes.includes(scope.split(":")[0] + ":*")
    );
    if (!hasScope) {
      reply.status(403).send({ error: "Insufficient permissions", required: requiredScopes });
    }
  };
}

export function requireRole(...roles: OrgRole[]) {
  return async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    if (!request.auth.role || !roles.includes(request.auth.role)) {
      reply.status(403).send({ error: "Insufficient role", required: roles });
    }
  };
}
```

**Testing**:
- `Unit: createApiKey returns a key starting with "dbaas_" and 76 chars total`
- `Unit: validateApiKey with valid key returns ApiKeyPayload with correct scopes`
- `Unit: validateApiKey with expired key returns null`
- `Unit: validateApiKey with revoked key returns null`
- `Unit: validateApiKey updates last_used_at timestamp`
- `Unit: authMiddleware with valid Bearer token sets request.auth`
- `Unit: authMiddleware with no auth header returns 401`
- `Unit: requireScope("databases:write") passes when scopes include "databases:write"`
- `Unit: requireScope("databases:write") passes when scopes include "databases:*"`
- `Unit: requireScope("databases:write") returns 403 when scopes are ["projects:read"]`
- `Integration (mocked Redis): session creation stores payload in Redis with TTL`
- `Integration (mocked Redis): session validation retrieves payload from Redis`
- `Integration (mocked Redis): expired session returns null`

---

#### 2.3 — Organization and Project CRUD Routes

**What**: Implement REST API endpoints for organization management (create, read, update, list members, invite) and project management (create, read, update, delete, list).

**Design**:

Organization endpoints:

| Method | Path | Description | Auth Scope |
|--------|------|-------------|------------|
| POST | /v1/organizations | Create organization | session |
| GET | /v1/organizations | List user's organizations | session / orgs:read |
| GET | /v1/organizations/:orgId | Get organization details | orgs:read |
| PATCH | /v1/organizations/:orgId | Update organization | orgs:write |
| GET | /v1/organizations/:orgId/members | List members | orgs:read |
| POST | /v1/organizations/:orgId/members | Invite member | orgs:write |
| PATCH | /v1/organizations/:orgId/members/:memberId | Update member role | orgs:write |
| DELETE | /v1/organizations/:orgId/members/:memberId | Remove member | orgs:write |

Project endpoints:

| Method | Path | Description | Auth Scope |
|--------|------|-------------|------------|
| POST | /v1/projects | Create project | projects:write |
| GET | /v1/projects | List projects in org | projects:read |
| GET | /v1/projects/:projectId | Get project details | projects:read |
| PATCH | /v1/projects/:projectId | Update project | projects:write |
| DELETE | /v1/projects/:projectId | Delete project (soft) | projects:write |

Request/response schemas:

```typescript
// Create organization request
interface CreateOrganizationRequest {
  name: string;           // min 2, max 255
  slug?: string;          // auto-generated from name if omitted; [a-z0-9-]
  billing_email?: string;
  plan?: "free" | "pro" | "team";
}

// Organization response
interface OrganizationResponse {
  id: string;
  name: string;
  slug: string;
  status: "active" | "suspended" | "deleted";
  plan: OrgPlan;
  settings: OrgSettings;
  member_count: number;
  created_at: string;     // ISO 8601
  updated_at: string;
}

// Create project request
interface CreateProjectRequest {
  name: string;
  slug?: string;
  description?: string;
  default_region: string;   // must be valid region ID
  settings?: Partial<ProjectSettings>;
}

// Project response
interface ProjectResponse {
  id: string;
  organization_id: string;
  name: string;
  slug: string;
  description: string | null;
  status: "active" | "paused" | "deleted";
  settings: ProjectSettings;
  database_count: number;
  created_by: string | null;
  created_at: string;
  updated_at: string;
}
```

Pagination follows RFC 8288 (Web Linking):

```typescript
// apps/api/src/lib/pagination.ts

interface PaginationParams {
  cursor?: string;       // opaque cursor (base64-encoded ID)
  limit?: number;        // default 20, max 100
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    has_more: boolean;
    next_cursor: string | null;
  };
}
```

**Testing**:
- `Unit: POST /v1/organizations with valid body creates org and returns 201`
- `Unit: POST /v1/organizations with duplicate slug returns 409`
- `Unit: POST /v1/organizations auto-generates slug from name ("My Org" -> "my-org")`
- `Unit: GET /v1/organizations lists only orgs the user is a member of`
- `Unit: PATCH /v1/organizations/:orgId by non-admin returns 403`
- `Unit: POST /v1/organizations/:orgId/members with valid email invites member`
- `Unit: POST /v1/organizations/:orgId/members with already-invited email returns 409`
- `Unit: POST /v1/projects with invalid region returns 400 with "invalid region" message`
- `Unit: POST /v1/projects with valid body creates project linked to org`
- `Unit: GET /v1/projects returns paginated list with cursor-based pagination`
- `Unit: GET /v1/projects?limit=5 returns at most 5 results with has_more flag`
- `Unit: DELETE /v1/projects/:projectId soft-deletes (sets status to "deleted")`
- `Integration (real DB): organization + project CRUD lifecycle end-to-end`

---

#### 2.4 — Audit Log Middleware

**What**: Create middleware that automatically logs all mutating API actions to the `audit_log` table.

**Design**:

```typescript
// apps/api/src/middleware/audit.ts

interface AuditEntry {
  organization_id: string;
  actor_id: string | null;
  actor_type: "user" | "api_key" | "system";
  action: string;           // "organization.create", "project.delete", "database.update"
  resource_type: string;    // "organization", "project", "database"
  resource_id: string;
  context: {
    ip_address: string;
    user_agent: string;
    request_id: string;
    api_key_id?: string;
    changes?: { before: unknown; after: unknown };
  };
}

export function auditPlugin(app: FastifyInstance): void {
  app.addHook("onResponse", async (request, reply) => {
    if (["POST", "PUT", "PATCH", "DELETE"].includes(request.method) && reply.statusCode < 400) {
      const entry: AuditEntry = {
        organization_id: request.auth?.organizationId,
        actor_id: request.auth?.userId ?? null,
        actor_type: request.auth?.type === "api_key" ? "api_key" : "user",
        action: `${request.routeOptions?.config?.resourceType}.${methodToAction(request.method)}`,
        resource_type: request.routeOptions?.config?.resourceType,
        resource_id: request.params?.id ?? reply.getHeader("x-resource-id"),
        context: {
          ip_address: request.ip,
          user_agent: request.headers["user-agent"] ?? "",
          request_id: request.id,
          api_key_id: request.auth?.type === "api_key" ? request.auth.keyId : undefined,
        },
      };
      await request.server.db.insert(auditLog).values(entry);
    }
  });
}

function methodToAction(method: string): string {
  switch (method) {
    case "POST": return "create";
    case "PUT": case "PATCH": return "update";
    case "DELETE": return "delete";
    default: return "unknown";
  }
}
```

**Testing**:
- `Unit: POST /v1/organizations creates an audit_log entry with action "organization.create"`
- `Unit: GET /v1/organizations does NOT create an audit_log entry`
- `Unit: audit entry contains correct ip_address, user_agent, and request_id`
- `Unit: audit entry for API key auth includes api_key_id in context`
- `Unit: failed requests (4xx, 5xx) do NOT create audit entries`
- `Integration (real DB): audit_log entries are queryable by organization_id and time range`

---

## Phase 3: Database Provisioning and Lifecycle

### Purpose

Implement the core value proposition: self-serve database provisioning. Users can create, list, update, and delete managed databases via the API. Database provisioning is handled asynchronously via BullMQ workers. After this phase, the platform can provision PostgreSQL instances (initially as Docker containers for development, with Kubernetes pod creation as the production path).

### Tasks

#### 3.1 — Database CRUD Endpoints

**What**: Implement REST API routes for database management with JSONB engine configuration and security settings.

**Design**:

| Method | Path | Description | Auth Scope |
|--------|------|-------------|------------|
| POST | /v1/databases | Create database | databases:write |
| GET | /v1/databases | List databases in project | databases:read |
| GET | /v1/databases/:databaseId | Get database details | databases:read |
| PATCH | /v1/databases/:databaseId | Update database config | databases:write |
| DELETE | /v1/databases/:databaseId | Delete database (async) | databases:write |

```typescript
// Create database request
interface CreateDatabaseRequest {
  project_id: string;
  name: string;
  engine?: DatabaseEngine;         // default: "postgresql"
  engine_version?: string;         // default: latest for engine
  region: string;                  // must be valid region
  engine_config?: Partial<EngineConfig>;
  security?: Partial<SecurityConfig>;
}

// Database response
interface DatabaseResponse {
  id: string;
  project_id: string;
  name: string;
  engine: DatabaseEngine;
  engine_version: string;
  region: string;
  cloud_provider: string;
  status: DatabaseStatus;
  storage_size_mb: number;
  engine_config: EngineConfig;
  security: SecurityConfig;
  connection_uri?: string;        // only when status is "available"
  created_by: string | null;
  created_at: string;
  updated_at: string;
}
```

Defaults applied on creation:

```typescript
const DEFAULT_SECURITY: SecurityConfig = {
  encryption_at_rest: true,
  encryption_algorithm: "AES-256",
  min_tls_version: "1.3",
  require_ssl: true,
};

const DEFAULT_PG_CONFIG: Partial<EngineConfig> = {
  max_connections: 100,
  shared_buffers: "256MB",
  work_mem: "4MB",
  wal_level: "replica",
  extensions_enabled: [],
  pg_bouncer: {
    pool_mode: "transaction",
    max_client_connections: 200,
    default_pool_size: 25,
  },
};
```

**Testing**:
- `Unit: POST /v1/databases with valid body returns 202 Accepted with status "provisioning"`
- `Unit: POST /v1/databases applies default security settings (encryption_at_rest: true, min_tls_version: "1.3")`
- `Unit: POST /v1/databases applies default engine_config for PostgreSQL`
- `Unit: POST /v1/databases with engine "mysql" applies MySQL default config`
- `Unit: POST /v1/databases validates region exists`
- `Unit: POST /v1/databases enforces plan limits (max_databases)`
- `Unit: GET /v1/databases filters by project_id query parameter`
- `Unit: DELETE /v1/databases/:id sets status to "deleting" and enqueues deletion job`
- `Integration (real DB): full create-read-update-delete lifecycle for a database`

---

#### 3.2 — Operations Tracking System

**What**: Implement the unified async operations tracker. All long-running tasks (provisioning, scaling, deletion, backup, restore) are represented as operations with status, progress, and error tracking.

**Design**:

```typescript
// apps/api/src/services/operations.service.ts

export class OperationsService {
  constructor(private db: DbClient) {}

  async create(params: {
    organizationId: string;
    resourceType: string;
    resourceId: string;
    operationType: string;
    input: Record<string, unknown>;
    startedBy?: string;
  }): Promise<string> {
    // Insert into operations table, return operation ID
  }

  async updateProgress(operationId: string, progressPct: number): Promise<void> {
    // Update progress_pct
  }

  async complete(operationId: string, output?: Record<string, unknown>): Promise<void> {
    // Set status = "completed", completed_at = now(), output
  }

  async fail(operationId: string, error: {
    code: string;
    message: string;
    retryable: boolean;
  }): Promise<void> {
    // Set status = "failed", error JSONB
  }

  async getByResource(resourceType: string, resourceId: string): Promise<Operation[]> {
    // List operations for a resource, ordered by created_at DESC
  }
}
```

Operations API:

| Method | Path | Description |
|--------|------|-------------|
| GET | /v1/operations | List operations (filterable by resource_type, status) |
| GET | /v1/operations/:operationId | Get operation details with progress |

**Testing**:
- `Unit: create() inserts operation with status "pending" and returns UUID`
- `Unit: updateProgress() updates progress_pct`
- `Unit: complete() sets status "completed" and completed_at timestamp`
- `Unit: fail() sets status "failed" with error JSONB containing code and message`
- `Unit: getByResource() returns operations ordered by created_at DESC`
- `Unit: GET /v1/operations?status=running returns only running operations`

---

#### 3.3 — BullMQ Worker Infrastructure and Provisioning Worker

**What**: Set up BullMQ queue infrastructure and implement the database provisioning worker that creates PostgreSQL instances.

**Design**:

```typescript
// apps/api/src/plugins/queue.ts

import { Queue, Worker, Job } from "bullmq";
import { Redis } from "ioredis";

export interface QueuePlugin {
  provisioningQueue: Queue;
  backupQueue: Queue;
  scalingQueue: Queue;
  cleanupQueue: Queue;
}

export function createQueues(redisConnection: Redis): QueuePlugin {
  return {
    provisioningQueue: new Queue("provisioning", { connection: redisConnection }),
    backupQueue: new Queue("backup", { connection: redisConnection }),
    scalingQueue: new Queue("scaling", { connection: redisConnection }),
    cleanupQueue: new Queue("cleanup", { connection: redisConnection }),
  };
}
```

```typescript
// apps/api/src/workers/provisioning.worker.ts

import { Worker, Job } from "bullmq";

interface ProvisionDatabaseJob {
  operationId: string;
  databaseId: string;
  engine: DatabaseEngine;
  engineVersion: string;
  region: string;
  engineConfig: EngineConfig;
  securityConfig: SecurityConfig;
}

export function createProvisioningWorker(deps: {
  redisConnection: Redis;
  db: DbClient;
  operations: OperationsService;
  kubernetes: KubernetesService;
}): Worker {
  return new Worker("provisioning", async (job: Job<ProvisionDatabaseJob>) => {
    const { operationId, databaseId, engine, engineVersion, region, engineConfig } = job.data;

    await deps.operations.updateProgress(operationId, 10);

    // Step 1: Create persistent volume claim for data
    await deps.kubernetes.createPVC(databaseId, region);
    await deps.operations.updateProgress(operationId, 30);

    // Step 2: Deploy database pod (PostgreSQL/MySQL container)
    const podInfo = await deps.kubernetes.deployDatabasePod({
      databaseId,
      engine,
      engineVersion,
      config: engineConfig,
      pvcName: `pvc-${databaseId}`,
    });
    await deps.operations.updateProgress(operationId, 60);

    // Step 3: Wait for pod ready
    await deps.kubernetes.waitForPodReady(podInfo.podName, { timeoutMs: 120_000 });
    await deps.operations.updateProgress(operationId, 80);

    // Step 4: Initialize database (create default role, set configs)
    const connectionUri = await initializeDatabase(podInfo, engineConfig);
    await deps.operations.updateProgress(operationId, 90);

    // Step 5: Update database status to "available"
    await deps.db.update(databases)
      .set({ status: "available", updatedAt: new Date() })
      .where(eq(databases.id, databaseId));

    // Step 6: Create default branch ("main")
    await deps.db.insert(branches).values({
      databaseId,
      name: "main",
      isDefault: true,
      isProtected: true,
      status: "ready",
    });

    await deps.operations.complete(operationId, { connectionUri });
  }, {
    connection: deps.redisConnection,
    concurrency: 5,
    limiter: { max: 10, duration: 60_000 },
  });
}
```

For development (no Kubernetes), a Docker-based provisioner:

```typescript
// apps/api/src/services/provisioning.service.ts

export interface ProvisionerBackend {
  createInstance(params: ProvisionParams): Promise<InstanceInfo>;
  deleteInstance(instanceId: string): Promise<void>;
  getStatus(instanceId: string): Promise<InstanceStatus>;
}

export class DockerProvisioner implements ProvisionerBackend {
  // Uses dockerode to create PostgreSQL containers for local dev
}

export class KubernetesProvisioner implements ProvisionerBackend {
  // Uses @kubernetes/client-node to create pods in production
}
```

**Testing**:
- `Unit: provisioning worker processes job and transitions database to "available"`
- `Unit: provisioning worker creates default "main" branch after database is ready`
- `Unit: provisioning worker calls operations.complete() with connectionUri on success`
- `Unit: provisioning worker calls operations.fail() on pod timeout`
- `Unit: provisioning worker updates progress at each step (10, 30, 60, 80, 90)`
- `Integration (mocked K8s): full provisioning flow from API request to "available" status`
- `Integration (Docker): DockerProvisioner creates a running PostgreSQL container`
- `Unit: concurrent provisioning respects limiter (max 10 per minute)`

---

#### 3.4 — Database Roles and Connection Management

**What**: Implement database role management (create, list, delete roles) and connection URI generation with password encryption.

**Design**:

```typescript
// Database roles API
// POST /v1/databases/:databaseId/roles
interface CreateRoleRequest {
  name: string;          // PostgreSQL role name, max 63 chars
  privileges?: {
    is_superuser?: boolean;
    can_login?: boolean;
    can_create_db?: boolean;
    connection_limit?: number;
    granted_roles?: string[];
  };
}

interface RoleResponse {
  id: string;
  database_id: string;
  name: string;
  privileges: RolePrivileges;
  connection_uri: string;    // postgresql://rolename:***@host:port/dbname
  created_at: string;
}
```

Connection URI generation:

```typescript
export function buildConnectionUri(params: {
  engine: DatabaseEngine;
  host: string;
  port: number;
  database: string;
  role: string;
  password: string;
  sslMode: string;
}): string {
  const protocol = params.engine === "postgresql" ? "postgresql" : "mysql";
  return `${protocol}://${encodeURIComponent(params.role)}:${encodeURIComponent(params.password)}@${params.host}:${params.port}/${params.database}?sslmode=${params.sslMode}`;
}
```

**Testing**:
- `Unit: POST /v1/databases/:id/roles creates role in control plane and on managed database`
- `Unit: role password is encrypted at rest and only returned on creation`
- `Unit: buildConnectionUri correctly encodes special characters in username/password`
- `Unit: buildConnectionUri uses "postgresql" protocol for PostgreSQL and "mysql" for MySQL`
- `Unit: DELETE /v1/databases/:id/roles/:roleId drops role on managed database`
- `Unit: cannot delete the last remaining role on a database (400 error)`

---

## Phase 4: Branching and Copy-on-Write

### Purpose

Implement instant database branching, the platform's key differentiator. Users can create branches from any point in a database's history, and branches share storage with their parent via copy-on-write semantics. After this phase, the platform supports the developer workflow of creating a branch per pull request and tearing it down on merge.

### Tasks

#### 4.1 — Branch CRUD Endpoints

**What**: Implement REST API routes for branch creation, listing, and deletion. Branch creation is async (enqueued as an operation).

**Design**:

| Method | Path | Description | Auth Scope |
|--------|------|-------------|------------|
| POST | /v1/databases/:databaseId/branches | Create branch | branches:write |
| GET | /v1/databases/:databaseId/branches | List branches | branches:read |
| GET | /v1/branches/:branchId | Get branch details | branches:read |
| PATCH | /v1/branches/:branchId | Update branch (name, protection) | branches:write |
| DELETE | /v1/branches/:branchId | Delete branch | branches:write |

```typescript
interface CreateBranchRequest {
  name: string;
  parent_branch_id?: string;     // defaults to default branch
  parent_timestamp?: string;     // ISO 8601; for PITR branching
  vcs_link?: {
    git_branch: string;
    pull_request_number?: number;
    pull_request_url?: string;
    auto_delete_on_merge?: boolean;
  };
}

interface BranchResponse {
  id: string;
  database_id: string;
  name: string;
  parent_branch_id: string | null;
  is_default: boolean;
  is_protected: boolean;
  status: BranchStatus;
  branch_point: BranchPoint | null;
  vcs_link: VcsLink | null;
  endpoint_count: number;
  created_by: string | null;
  created_at: string;
  updated_at: string;
}
```

Branch creation logic:

```typescript
// apps/api/src/services/branching.service.ts

export class BranchingService {
  async createBranch(params: {
    databaseId: string;
    name: string;
    parentBranchId?: string;
    parentTimestamp?: Date;
    vcsLink?: VcsLink;
    createdBy?: string;
  }): Promise<{ branchId: string; operationId: string }> {
    // 1. Resolve parent branch (default if not specified)
    // 2. Validate plan limits (max_branches_per_db)
    // 3. Insert branch record with status "creating"
    // 4. Enqueue branching job
    // 5. Return branch ID and operation ID
  }

  async deleteBranch(branchId: string): Promise<void> {
    // 1. Validate branch is not default or protected
    // 2. Delete associated endpoints
    // 3. Soft-delete branch
    // 4. Enqueue cleanup job
  }
}
```

**Testing**:
- `Unit: POST creates branch with status "creating" and returns 202`
- `Unit: POST with parent_branch_id forks from specified parent`
- `Unit: POST without parent_branch_id forks from default branch`
- `Unit: POST with parent_timestamp creates PITR branch`
- `Unit: POST enforces plan limit on max_branches_per_db`
- `Unit: DELETE on default branch returns 400 "Cannot delete default branch"`
- `Unit: DELETE on protected branch returns 400 "Cannot delete protected branch"`
- `Unit: GET /v1/databases/:id/branches returns all branches for database`
- `Unit: PATCH /v1/branches/:id can rename branch`
- `Unit: PATCH /v1/branches/:id can toggle is_protected`

---

#### 4.2 — Branching Worker (Copy-on-Write)

**What**: Implement the async worker that performs the actual branch creation using copy-on-write storage semantics.

**Design**:

```typescript
interface CreateBranchJob {
  operationId: string;
  branchId: string;
  databaseId: string;
  parentBranchId: string;
  parentLsn?: string;
  parentTimestamp?: string;
}

// Worker flow:
// 1. Get parent branch's storage location
// 2. Create CoW snapshot of parent's data directory
// 3. If parentTimestamp specified, replay WAL to that point
// 4. Record branch point metadata (LSN, timestamp, storage path)
// 5. Update branch status to "ready"
// 6. Auto-create a compute endpoint for the new branch
```

Branch point metadata stored in JSONB:

```typescript
// Stored in branches.branch_point
const branchPoint: BranchPoint = {
  parent_lsn: "0/1A2B3C4D",
  parent_timestamp: "2026-05-25T10:30:00Z",
  cow_storage_path: "s3://dbaas-storage/branches/br_abc123/",
  data_size_at_fork_mb: 2048,
};
```

**Testing**:
- `Unit: worker creates CoW snapshot and records branch_point metadata`
- `Unit: worker transitions branch status from "creating" to "ready"`
- `Unit: worker auto-creates compute endpoint for new branch`
- `Unit: worker with parentTimestamp replays WAL to specified point`
- `Unit: worker failure transitions branch to "error" and operation to "failed"`
- `Integration (mocked storage): full branch creation from parent with data verification`
- `Unit: branch creation completes in under 5 seconds for any parent data size (CoW, not copy)`

---

#### 4.3 — Compute Endpoint Management

**What**: Implement compute endpoint CRUD with scale-to-zero support and autoscaling configuration.

**Design**:

| Method | Path | Description |
|--------|------|-------------|
| POST | /v1/branches/:branchId/endpoints | Create endpoint |
| GET | /v1/branches/:branchId/endpoints | List endpoints |
| GET | /v1/endpoints/:endpointId | Get endpoint details |
| PATCH | /v1/endpoints/:endpointId | Update compute config |
| DELETE | /v1/endpoints/:endpointId | Delete endpoint |
| POST | /v1/endpoints/:endpointId/suspend | Suspend endpoint |
| POST | /v1/endpoints/:endpointId/resume | Resume endpoint |

```typescript
interface CreateEndpointRequest {
  type?: EndpointType;           // default: "read_write"
  compute?: Partial<ComputeConfig>;
  connection_pooling?: Partial<ConnectionPoolingConfig>;
}

interface EndpointResponse {
  id: string;
  branch_id: string;
  host: string;
  port: number;
  type: EndpointType;
  status: EndpointStatus;
  compute: ComputeConfig;
  connection_pooling: ConnectionPoolingConfig | null;
  connection_uri: string;
  pooler_connection_uri: string | null;
  last_active_at: string | null;
  created_at: string;
  updated_at: string;
}
```

Hostname generation:

```typescript
export function generateEndpointHost(branchName: string, region: string): string {
  const slug = branchName.toLowerCase().replace(/[^a-z0-9]/g, "-").slice(0, 20);
  const suffix = crypto.randomBytes(4).toString("hex");
  return `ep-${slug}-${suffix}.${region}.dbaas.example.com`;
}
// Example: "ep-feature-user-prof-a1b2c3d4.us-east-1.dbaas.example.com"
```

**Testing**:
- `Unit: POST creates endpoint with generated hostname and status "creating"`
- `Unit: generateEndpointHost produces valid DNS hostname`
- `Unit: PATCH /v1/endpoints/:id updates compute config (min_cu, max_cu, scale_to_zero)`
- `Unit: POST /v1/endpoints/:id/suspend transitions from "active" to "suspended"`
- `Unit: POST /v1/endpoints/:id/resume transitions from "idle"/"suspended" to "active"`
- `Unit: endpoint response includes connection_uri when status is "active"`
- `Unit: endpoint response includes pooler_connection_uri when connection_pooling is enabled`
- `Unit: DELETE endpoint removes it from branch`
- `Unit: cannot create more than one read_write endpoint per branch`

---

## Phase 5: Backup, Restore, and Point-in-Time Recovery

### Purpose

Implement automated and manual backup capabilities with point-in-time recovery (PITR). After this phase, databases are automatically backed up on schedule, users can create manual backups, and data can be restored to any point within the retention window.

### Tasks

#### 5.1 — Backup CRUD and Scheduling

**What**: Implement backup creation (manual and scheduled), listing, and deletion endpoints. Scheduled backups run via BullMQ repeatable jobs.

**Design**:

| Method | Path | Description |
|--------|------|-------------|
| POST | /v1/databases/:databaseId/backups | Create manual backup |
| GET | /v1/databases/:databaseId/backups | List backups |
| GET | /v1/backups/:backupId | Get backup details |
| DELETE | /v1/backups/:backupId | Delete backup |

```typescript
interface CreateBackupRequest {
  type?: "manual" | "pre_migration";
  // "scheduled" is system-created only
}

interface BackupResponse {
  id: string;
  branch_id: string;
  type: "scheduled" | "manual" | "pre_migration";
  status: "in_progress" | "completed" | "failed" | "expired";
  details: {
    storage_path: string;
    size_bytes: number;
    wal_position: string;
    compression: string;
    duration_seconds: number;
    tables_count: number;
  };
  started_at: string;
  completed_at: string | null;
  expires_at: string | null;
  created_at: string;
}
```

Backup scheduling via BullMQ repeatable jobs:

```typescript
// Schedule daily backup for each database at creation time
await backupQueue.add(
  `backup:${databaseId}`,
  { databaseId, branchId: defaultBranchId, type: "scheduled" },
  {
    repeat: { pattern: "0 2 * * *" },  // 2 AM daily
    jobId: `scheduled-backup:${databaseId}`,
  },
);
```

**Testing**:
- `Unit: POST /v1/databases/:id/backups creates manual backup with status "in_progress"`
- `Unit: GET /v1/databases/:id/backups returns backups ordered by created_at DESC`
- `Unit: scheduled backup job runs at configured interval`
- `Unit: expired backups are cleaned up by the cleanup worker`
- `Unit: backup details JSONB includes size_bytes, wal_position, compression`
- `Integration (mocked S3): backup worker uploads backup to S3-compatible storage`

---

#### 5.2 — Backup Worker

**What**: Implement the async backup worker that creates PostgreSQL base backups and uploads them to S3-compatible storage.

**Design**:

```typescript
interface BackupJob {
  operationId: string;
  backupId: string;
  databaseId: string;
  branchId: string;
  type: "scheduled" | "manual" | "pre_migration";
}

// Worker flow:
// 1. Connect to managed database instance
// 2. Run pg_basebackup (or logical backup via pg_dump for smaller DBs)
// 3. Compress with zstd
// 4. Encrypt with AES-256-GCM using database's KMS key
// 5. Upload to S3: s3://dbaas-backups/{org_id}/{db_id}/{backup_id}.tar.zst.enc
// 6. Record WAL position at backup time
// 7. Update backup status and details JSONB
// 8. Set expires_at based on plan's backup_retention_days
```

**Testing**:
- `Unit: worker creates compressed, encrypted backup file`
- `Unit: worker records correct WAL position in backup details`
- `Unit: worker sets expires_at = now() + retention_days from org plan`
- `Unit: worker transitions backup status from "in_progress" to "completed"`
- `Unit: worker failure sets status "failed" with error message in details`
- `Integration (mocked S3): backup uploaded to correct S3 path with correct content type`
- `Unit: backup worker respects concurrency limit (max 3 concurrent backups)`

---

#### 5.3 — Restore Operations and PITR

**What**: Implement restore-from-backup and point-in-time recovery, creating new branches from restored data.

**Design**:

```typescript
// POST /v1/backups/:backupId/restore
interface RestoreFromBackupRequest {
  target_branch_name: string;
}

// POST /v1/databases/:databaseId/restore
interface PITRRestoreRequest {
  target_timestamp: string;       // ISO 8601
  target_branch_name: string;
}

// Restore creates a new branch from the restored data
// Flow:
// 1. Validate target_timestamp is within retention window
// 2. Find nearest backup before target_timestamp
// 3. Create new branch record with status "creating"
// 4. Enqueue restore job
// 5. Restore job: download backup, decompress, decrypt, apply
// 6. For PITR: replay WAL from backup LSN to target_timestamp
// 7. Create compute endpoint for restored branch
// 8. Update branch status to "ready"
```

**Testing**:
- `Unit: POST /v1/backups/:id/restore creates new branch from backup`
- `Unit: POST /v1/databases/:id/restore with valid timestamp creates PITR branch`
- `Unit: restore with timestamp outside retention window returns 400`
- `Unit: restore finds the nearest backup before the target timestamp`
- `Unit: restored branch has its own compute endpoint`
- `Integration (mocked storage): full backup-restore cycle verifies data integrity`

---

## Phase 6: Management Console (Next.js)

### Purpose

Build the web-based management console for visual database management. After this phase, users can sign up, manage organizations, provision databases, create branches, view backups, and monitor endpoint status through a browser-based dashboard.

### Tasks

#### 6.1 — Console Authentication (Login, Signup, OAuth)

**What**: Implement the Next.js authentication flow with email/password signup, login, and GitHub/Google OAuth.

**Design**:

```typescript
// apps/console/src/app/(auth)/login/page.tsx
// - Email + password form
// - "Sign in with GitHub" button
// - "Sign in with Google" button
// - Link to signup page

// apps/console/src/app/(auth)/signup/page.tsx
// - Name, email, password form
// - Email verification flow

// apps/console/src/lib/api-client.ts
export class ApiClient {
  constructor(private baseUrl: string, private sessionId?: string) {}

  async login(email: string, password: string): Promise<SessionResponse> { /* ... */ }
  async signup(name: string, email: string, password: string): Promise<void> { /* ... */ }
  async oauthCallback(provider: string, code: string): Promise<SessionResponse> { /* ... */ }

  // Resource methods
  async listOrganizations(): Promise<PaginatedResponse<OrganizationResponse>> { /* ... */ }
  async createDatabase(params: CreateDatabaseRequest): Promise<DatabaseResponse> { /* ... */ }
  // ... all other resource methods
}
```

**Testing**:
- `E2E: signup flow creates account and redirects to dashboard`
- `E2E: login with valid credentials redirects to dashboard`
- `E2E: login with invalid credentials shows error message`
- `E2E: GitHub OAuth flow completes and creates session`
- `Unit: ApiClient includes session cookie in all requests`
- `Unit: expired session redirects to login page`

---

#### 6.2 — Dashboard Layout and Navigation

**What**: Build the dashboard layout with sidebar navigation, organization switcher, and responsive design.

**Design**:

Navigation structure:
```
Sidebar:
├── [Org Switcher]
├── Overview (dashboard home)
├── Projects
│   └── [project] → Databases
│       └── [database]
│           ├── Overview
│           ├── Branches
│           ├── Backups
│           ├── Roles
│           └── Settings
├── API Keys
├── Members
├── Billing
└── Settings
```

Key components:
- `<OrgSwitcher>` — dropdown to switch between organizations
- `<Sidebar>` — collapsible sidebar with navigation tree
- `<BreadcrumbNav>` — breadcrumb trail showing org > project > database > branch
- `<StatusBadge>` — color-coded badge for resource status (green=available, yellow=scaling, red=error)

**Testing**:
- `E2E: sidebar shows correct navigation items for current user's role`
- `E2E: org switcher lists all user's organizations`
- `E2E: breadcrumb navigation reflects current resource path`
- `Unit: StatusBadge renders correct color for each status`
- `E2E: sidebar collapses on mobile viewport`

---

#### 6.3 — Database and Branch Management UI

**What**: Build the database list, database detail, branch list, and branch creation views.

**Design**:

Database list page (`/projects/:projectId/databases`):
- Table view with columns: Name, Engine, Region, Status, Storage, Branches, Created
- "Create Database" button opening a modal form
- Status badges with real-time updates via SSE

Database detail page (`/databases/:databaseId`):
- Overview tab: connection info, engine config, storage metrics
- Branches tab: branch list with lineage visualization
- Backups tab: backup list with restore actions
- Roles tab: role management
- Settings tab: engine config, security settings

Branch creation modal:
- Parent branch selector (dropdown with branch names)
- Branch name input (auto-suggested from git branch if VCS linked)
- Optional: fork from specific timestamp (date-time picker for PITR)

Branch visualization:
- Tree/graph view showing branch lineage (parent → child relationships)
- Each node shows branch name, status, and endpoint count

**Testing**:
- `E2E: database list shows all databases in a project`
- `E2E: "Create Database" modal submits form and shows provisioning status`
- `E2E: database detail page shows connection URI for available databases`
- `E2E: branch tab shows branch tree with correct parent-child relationships`
- `E2E: "Create Branch" modal creates branch and shows in branch list`
- `E2E: branch deletion shows confirmation dialog and removes from list`

---

#### 6.4 — Real-Time Status Updates (SSE)

**What**: Implement Server-Sent Events for real-time updates of database, branch, and endpoint status changes.

**Design**:

```typescript
// apps/api/src/routes/events/sse.ts

// GET /v1/events/stream?project_id=xxx
// SSE endpoint that streams status changes for resources in a project

interface StatusChangeEvent {
  event: "status_change";
  data: {
    resource_type: "database" | "branch" | "endpoint" | "backup" | "operation";
    resource_id: string;
    old_status: string;
    new_status: string;
    timestamp: string;
  };
}

// Backend: use Redis pub/sub to broadcast status changes
// When any resource status changes:
//   redis.publish(`project:${projectId}:events`, JSON.stringify(event))
// SSE handler subscribes to the channel and streams to connected clients
```

```typescript
// apps/console/src/lib/hooks/use-status-stream.ts

export function useStatusStream(projectId: string) {
  const [events, setEvents] = useState<StatusChangeEvent[]>([]);

  useEffect(() => {
    const source = new EventSource(`/api/events/stream?project_id=${projectId}`);
    source.addEventListener("status_change", (e) => {
      const event = JSON.parse(e.data) as StatusChangeEvent;
      setEvents((prev) => [...prev.slice(-100), event]);
      // Trigger React Query cache invalidation for the affected resource
    });
    return () => source.close();
  }, [projectId]);

  return events;
}
```

**Testing**:
- `Integration: SSE endpoint streams status_change events when database status changes`
- `Integration: SSE endpoint filters events by project_id`
- `Unit: useStatusStream hook connects to SSE endpoint and receives events`
- `Unit: SSE connection reconnects automatically on disconnect`
- `E2E: creating a database shows real-time status progression (provisioning → available)`

---

## Phase 7: CLI Tool

### Purpose

Build the `dbaas` CLI tool for developer workflow operations. After this phase, developers can manage databases, branches, and backups from the terminal, authenticate with API keys, and integrate database operations into CI/CD pipelines.

### Tasks

#### 7.1 — CLI Scaffold and Authentication

**What**: Create the CLI entry point with auth management (login, API key configuration), output formatting, and configuration file handling.

**Design**:

```typescript
// apps/cli/src/index.ts

import { Command } from "commander";

const program = new Command()
  .name("dbaas")
  .description("Database-as-a-Service Platform CLI")
  .version("1.0.0");

program
  .command("auth")
  .description("Manage authentication")
  .addCommand(new Command("login").description("Login via browser").action(authLogin))
  .addCommand(new Command("token").description("Set API key").argument("<token>").action(authToken))
  .addCommand(new Command("status").description("Show auth status").action(authStatus));

// Config file: ~/.config/dbaas/config.json
interface CliConfig {
  api_url: string;           // default: "https://api.dbaas.example.com"
  auth_token?: string;       // API key
  default_project?: string;  // project ID for convenience
  output_format: "table" | "json" | "yaml";
}
```

Output formatting:

```typescript
// apps/cli/src/output.ts

export function formatOutput(data: unknown, format: "table" | "json" | "yaml"): string {
  switch (format) {
    case "json": return JSON.stringify(data, null, 2);
    case "yaml": return yaml.dump(data);
    case "table": return formatTable(data);
  }
}
```

**Testing**:
- `Unit: dbaas --version prints version number`
- `Unit: dbaas --help lists all commands`
- `Unit: dbaas auth token <key> stores key in config file`
- `Unit: dbaas auth status shows current auth state`
- `Unit: config file is created in ~/.config/dbaas/ with correct permissions (0600)`
- `Unit: --output json formats response as JSON`
- `Unit: --output table formats response as aligned table`

---

#### 7.2 — Database and Branch CLI Commands

**What**: Implement CLI commands for database and branch operations.

**Design**:

```
dbaas database list [--project <id>] [--output json|table|yaml]
dbaas database create --name <name> --project <id> --region <region> [--engine postgresql|mysql]
dbaas database get <database-id>
dbaas database delete <database-id> [--force]

dbaas branch list --database <id>
dbaas branch create --database <id> --name <name> [--parent <branch-id>] [--timestamp <iso8601>]
dbaas branch get <branch-id>
dbaas branch delete <branch-id>

dbaas endpoint list --branch <id>
dbaas endpoint get <endpoint-id>
dbaas endpoint suspend <endpoint-id>
dbaas endpoint resume <endpoint-id>

dbaas backup list --database <id>
dbaas backup create --database <id>
dbaas backup restore <backup-id> --branch-name <name>

dbaas connect <database-id> [--branch <name>]
  # Opens psql/mysql shell connected to the database
```

**Testing**:
- `E2E: dbaas database list returns formatted table of databases`
- `E2E: dbaas database create provisions database and shows operation status`
- `E2E: dbaas branch create creates branch and displays branch ID`
- `E2E: dbaas connect opens psql session to the correct endpoint`
- `Unit: dbaas database delete without --force prompts for confirmation`
- `Unit: dbaas database delete --force skips confirmation`
- `Unit: all commands return non-zero exit code on API errors`
- `Unit: all commands respect --output format flag`

---

## Phase 8: Observability and Monitoring

### Purpose

Implement the observability stack: Prometheus metrics export from managed databases and the control plane, OpenTelemetry tracing for API requests and worker operations, structured logging, and alerting rules. After this phase, operators have full visibility into platform health and database performance.

### Tasks

#### 8.1 — Prometheus Metrics Collection

**What**: Implement metrics collection from managed database instances and expose a Prometheus-compatible `/metrics` endpoint on the control plane.

**Design**:

Control plane metrics:

```typescript
// apps/api/src/lib/metrics.ts

import { Registry, Counter, Histogram, Gauge } from "prom-client";

export const registry = new Registry();

export const httpRequestDuration = new Histogram({
  name: "dbaas_http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [registry],
});

export const databaseCount = new Gauge({
  name: "dbaas_databases_total",
  help: "Total number of databases by status",
  labelNames: ["status", "engine"],
  registers: [registry],
});

export const branchCount = new Gauge({
  name: "dbaas_branches_total",
  help: "Total number of branches by status",
  labelNames: ["status"],
  registers: [registry],
});

export const operationDuration = new Histogram({
  name: "dbaas_operation_duration_seconds",
  help: "Async operation duration in seconds",
  labelNames: ["operation_type", "status"],
  buckets: [1, 5, 10, 30, 60, 120, 300],
  registers: [registry],
});

export const backupSizeBytes = new Histogram({
  name: "dbaas_backup_size_bytes",
  help: "Backup size in bytes",
  labelNames: ["type"],
  buckets: [1e6, 1e7, 1e8, 1e9, 1e10],
  registers: [registry],
});
```

Managed database metrics (collected per endpoint):

```typescript
// Metrics collected from pg_stat_statements, pg_stat_activity, pg_stat_bgwriter
interface DatabaseMetrics {
  connections_active: number;
  connections_idle: number;
  transactions_committed: number;
  transactions_rolled_back: number;
  rows_fetched: number;
  rows_inserted: number;
  rows_updated: number;
  rows_deleted: number;
  cache_hit_ratio: number;
  disk_usage_bytes: number;
  replication_lag_seconds: number;
}
```

**Testing**:
- `Unit: GET /metrics returns Prometheus text format`
- `Unit: httpRequestDuration records correct duration for API requests`
- `Unit: databaseCount gauge reflects current database count by status`
- `Unit: managed database metrics collector gathers pg_stat_* data`
- `Integration: Prometheus can scrape /metrics endpoint and parse all metric families`

---

#### 8.2 — OpenTelemetry Tracing

**What**: Instrument the API server and workers with OpenTelemetry distributed tracing.

**Design**:

```typescript
// apps/api/src/lib/tracing.ts

import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { FastifyInstrumentation } from "@opentelemetry/instrumentation-fastify";
import { PgInstrumentation } from "@opentelemetry/instrumentation-pg";
import { IORedisInstrumentation } from "@opentelemetry/instrumentation-ioredis";

export function initTracing(serviceName: string, otlpEndpoint?: string): NodeSDK {
  const sdk = new NodeSDK({
    serviceName,
    traceExporter: otlpEndpoint
      ? new OTLPTraceExporter({ url: otlpEndpoint })
      : undefined,
    instrumentations: [
      new FastifyInstrumentation(),
      new PgInstrumentation(),
      new IORedisInstrumentation(),
    ],
  });
  sdk.start();
  return sdk;
}
```

Custom spans for business operations:

```typescript
// Provisioning worker adds spans:
// dbaas.provision.create_pvc
// dbaas.provision.deploy_pod
// dbaas.provision.wait_ready
// dbaas.provision.init_database
// dbaas.provision.create_default_branch
```

**Testing**:
- `Unit: initTracing creates SDK with correct service name and exporter`
- `Unit: API requests generate trace spans with correct attributes`
- `Unit: provisioning worker creates child spans for each step`
- `Unit: trace_id is propagated from API request to worker job`
- `Integration: traces appear in OTLP collector with correct parent-child relationships`

---

#### 8.3 — Alert Rules API

**What**: Implement alert rule management so users can configure threshold-based alerts on database metrics.

**Design**:

```typescript
// POST /v1/projects/:projectId/alerts
interface CreateAlertRuleRequest {
  name: string;
  metric: string;                // "connections_active", "cache_hit_ratio", "disk_usage_bytes"
  condition: "gt" | "lt" | "gte" | "lte" | "eq";
  threshold: number;
  evaluation_period_seconds?: number;   // default: 300
  notification_channels: Array<
    | { type: "email"; target: string }
    | { type: "slack"; webhook_url: string }
    | { type: "webhook"; url: string; headers?: Record<string, string> }
  >;
}
```

**Testing**:
- `Unit: POST /v1/projects/:id/alerts creates alert rule`
- `Unit: alert evaluator triggers notification when threshold exceeded`
- `Unit: alert evaluator respects evaluation_period (no alert for transient spikes)`
- `Unit: notification dispatch sends to correct channel type (email, Slack, webhook)`
- `Unit: GET /v1/projects/:id/alerts returns alert rules with last_triggered_at`

---

## Phase 9: VCS Integration and Branch Lifecycle Automation

### Purpose

Integrate with GitHub and GitLab to automate database branch lifecycle management. After this phase, creating a pull request automatically provisions a database branch, and merging the PR automatically cleans up the branch.

### Tasks

#### 9.1 — GitHub App Webhook Handler

**What**: Implement GitHub App installation and webhook handlers for pull request events.

**Design**:

```typescript
// Webhook events handled:
// - installation.created → store installation_id for org
// - pull_request.opened → create database branch
// - pull_request.closed (merged) → delete database branch
// - pull_request.reopened → recreate database branch

// POST /v1/webhooks/github
// Validates X-Hub-Signature-256 header
// Routes to event-specific handlers

interface GithubWebhookPayload {
  action: string;
  installation: { id: number };
  repository: { full_name: string; owner: { login: string } };
  pull_request?: {
    number: number;
    head: { ref: string };       // branch name
    merged: boolean;
    html_url: string;
  };
}
```

Branch naming convention for auto-created branches:
```
pr-{pr_number}/{git_branch_name}
Example: pr-142/feature-user-profiles
```

**Testing**:
- `Integration (mocked): PR opened webhook creates database branch with vcs_link`
- `Integration (mocked): PR closed+merged webhook deletes branch with auto_delete_on_merge=true`
- `Integration (mocked): PR closed+not-merged does NOT delete branch`
- `Integration (mocked): PR reopened recreates branch from parent`
- `Unit: webhook signature validation rejects invalid signatures`
- `Unit: webhook signature validation accepts valid HMAC-SHA256 signatures`
- `Unit: auto-created branch name follows "pr-{number}/{ref}" convention`

---

#### 9.2 — GitLab Webhook Handler

**What**: Implement GitLab webhook handler for merge request events using the same branching service.

**Design**:

```typescript
// POST /v1/webhooks/gitlab
// Validates X-Gitlab-Token header

// Events handled:
// - merge_request (open) → create database branch
// - merge_request (merge) → delete database branch
// - merge_request (reopen) → recreate database branch
```

**Testing**:
- `Integration (mocked): MR opened webhook creates database branch`
- `Integration (mocked): MR merged webhook deletes branch`
- `Unit: GitLab token validation rejects invalid tokens`
- `Unit: branch naming is consistent between GitHub and GitLab handlers`

---

#### 9.3 — VCS Integration Settings UI

**What**: Add VCS integration configuration to the project settings page in the console.

**Design**:

Project settings page additions:
- "Connect to GitHub" button (GitHub App installation flow)
- "Connect to GitLab" button (webhook URL + token generation)
- Repository selector (list repos from installed GitHub App)
- Toggle: "Auto-create branches for pull requests"
- Toggle: "Auto-delete branches on merge"
- Display: current VCS link status for each database branch

**Testing**:
- `E2E: GitHub App installation flow connects repo to project`
- `E2E: GitLab webhook configuration shows webhook URL and generated token`
- `E2E: toggling auto-create/auto-delete updates project settings`
- `E2E: branch list shows VCS link status (PR number, URL) for linked branches`

---

## Phase 10: AI-Assisted Database Operations

### Purpose

Implement AI-powered features: schema and index recommendations from query pattern analysis, natural-language query interface, and the MCP server endpoint. After this phase, the platform's AI-native advantage is operational.

### Tasks

#### 10.1 — Query Pattern Collector

**What**: Implement background collection of query patterns from managed databases using `pg_stat_statements`.

**Design**:

```typescript
// apps/api/src/workers/ai-analysis.worker.ts

// Runs periodically (every 15 minutes) per database
// 1. Connect to managed database
// 2. Query pg_stat_statements for top-N queries by total_time
// 3. Normalize queries (replace literals with $N placeholders)
// 4. Compute query fingerprint (SHA-256 of normalized query)
// 5. Upsert into query_patterns tracking table
// 6. Feed patterns to AI recommendation engine

interface QueryPattern {
  database_id: string;
  query_fingerprint: string;
  sample_query: string;
  call_count: number;
  total_time_ms: number;
  mean_time_ms: number;
  max_time_ms: number;
  rows_returned: number;
}
```

Store as part of the `ai_recommendations` table in the control plane, separate from the managed database:

```typescript
// ai_recommendations JSONB recommendation field for index suggestion:
{
  description: "Adding index on orders(tenant_id, created_at DESC) reduces scan time by ~60%",
  suggested_sql: "CREATE INDEX CONCURRENTLY idx_orders_tenant_created ON orders(tenant_id, created_at DESC);",
  estimated_impact: {
    query_time_reduction_pct: 60,
    affected_queries: 12,
    storage_increase_mb: 45
  },
  analysis: {
    sample_queries: ["SELECT * FROM orders WHERE tenant_id = $1 ORDER BY created_at DESC LIMIT 20"],
    current_plan: "Seq Scan on orders (cost=0.00..18234.00 rows=500 width=128)",
    projected_plan: "Index Scan using idx_orders_tenant_created (cost=0.43..24.50 rows=500 width=128)"
  }
}
```

**Testing**:
- `Unit: query normalizer replaces literal values with $N placeholders`
- `Unit: query fingerprint is deterministic for equivalent queries`
- `Unit: collector upserts patterns (increments call_count, updates totals)`
- `Integration (real PG): collector reads pg_stat_statements from a test database`
- `Unit: collector runs every 15 minutes per database`

---

#### 10.2 — AI Recommendation Engine

**What**: Implement the AI engine that generates index, schema, and query optimization recommendations from collected patterns.

**Design**:

```typescript
// apps/api/src/services/ai-recommendations.service.ts

export class AIRecommendationService {
  constructor(
    private db: DbClient,
    private llmClient: AnthropicClient,  // Claude API
  ) {}

  async analyzeDatabase(databaseId: string): Promise<void> {
    // 1. Fetch query patterns for database
    // 2. Fetch current schema (from managed database via pg_catalog)
    // 3. Fetch existing indexes
    // 4. Send to LLM with structured prompt
    // 5. Parse LLM response into typed recommendations
    // 6. Insert recommendations with status "pending"
  }

  async applyRecommendation(recommendationId: string, userId: string): Promise<void> {
    // 1. Fetch recommendation and its suggested_sql
    // 2. Execute SQL on managed database
    // 3. Update recommendation status to "applied"
    // 4. Log in audit trail
  }
}
```

LLM prompt structure:

```typescript
const systemPrompt = `You are a PostgreSQL performance expert. Analyze the following query patterns, schema, and existing indexes to generate actionable recommendations.

For each recommendation, provide:
1. Type: "index", "schema", or "query_rewrite"
2. Severity: "critical", "warning", or "info"
3. Title: concise description (max 100 chars)
4. Description: detailed explanation
5. Suggested SQL: executable DDL or rewritten query
6. Estimated impact: quantified improvement

Output as JSON array matching the RecommendationOutput schema.`;

interface RecommendationOutput {
  type: "index" | "schema" | "query_rewrite";
  severity: "critical" | "warning" | "info";
  title: string;
  description: string;
  suggested_sql: string;
  estimated_impact: {
    query_time_reduction_pct?: number;
    storage_increase_mb?: number;
    affected_queries?: number;
  };
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | /v1/databases/:id/recommendations | List recommendations |
| POST | /v1/databases/:id/recommendations/generate | Trigger analysis |
| POST | /v1/recommendations/:id/apply | Apply recommendation |
| POST | /v1/recommendations/:id/dismiss | Dismiss recommendation |

**Testing**:
- `Unit: analyzeDatabase fetches patterns, schema, and indexes before calling LLM`
- `Unit: LLM response is parsed into typed RecommendationOutput array`
- `Unit: invalid LLM response is handled gracefully (logged, not stored)`
- `Unit: applyRecommendation executes suggested_sql on managed database`
- `Unit: applyRecommendation updates status to "applied" with applied_by and applied_at`
- `Integration (mocked LLM): full cycle from patterns to stored recommendations`
- `Unit: POST /v1/recommendations/:id/dismiss sets status to "dismissed"`

---

#### 10.3 — Natural-Language Query Interface

**What**: Implement a natural-language-to-SQL interface in the console that allows users to query databases using plain English.

**Design**:

```typescript
// POST /v1/databases/:databaseId/nl-query
interface NLQueryRequest {
  question: string;              // "Show me the top 10 customers by total order value"
  branch_id?: string;            // which branch to query (defaults to default branch)
  explain_only?: boolean;        // return SQL without executing (default: false)
}

interface NLQueryResponse {
  question: string;
  generated_sql: string;
  explanation: string;           // LLM explanation of what the SQL does
  results?: Array<Record<string, unknown>>;   // query results (if not explain_only)
  columns?: Array<{ name: string; type: string }>;
  row_count?: number;
  execution_time_ms?: number;
}
```

Safety constraints:
- Only SELECT queries are generated (no DDL, DML)
- Query timeout of 30 seconds enforced
- Results capped at 1000 rows
- Read-only database role used for execution

**Testing**:
- `Unit: NL query "count all users" generates valid SELECT COUNT(*) SQL`
- `Unit: generated SQL is always a SELECT statement (never INSERT, UPDATE, DELETE, DROP)`
- `Unit: explain_only mode returns SQL without executing`
- `Unit: query execution uses read-only role`
- `Unit: query timeout terminates long-running queries after 30 seconds`
- `Unit: results are capped at 1000 rows with truncation indicator`
- `Integration (mocked LLM): full flow from question to results`

---

#### 10.4 — MCP Server

**What**: Implement the Model Context Protocol server that exposes database operations as tools for AI coding assistants.

**Design**:

```typescript
// apps/mcp-server/src/index.ts

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "dbaas-platform",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {},
    resources: {},
  },
});

// Tools exposed:
// - dbaas_list_databases: List databases in a project
// - dbaas_get_schema: Introspect database schema (tables, columns, types)
// - dbaas_execute_query: Run a read-only SQL query
// - dbaas_create_branch: Create a new database branch
// - dbaas_list_branches: List branches for a database
// - dbaas_get_recommendations: Get AI recommendations for a database
// - dbaas_apply_recommendation: Apply a recommendation

// Resources exposed:
// - dbaas://databases/{id}/schema: Current database schema as text
// - dbaas://databases/{id}/stats: Database statistics summary
```

Tool definitions:

```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "dbaas_get_schema",
      description: "Retrieve the schema of a managed database including tables, columns, types, indexes, and constraints",
      inputSchema: {
        type: "object",
        properties: {
          database_id: { type: "string", description: "Database UUID" },
          branch: { type: "string", description: "Branch name (default: main)" },
          table_filter: { type: "string", description: "Optional regex filter for table names" },
        },
        required: ["database_id"],
      },
    },
    {
      name: "dbaas_execute_query",
      description: "Execute a read-only SQL query against a managed database branch",
      inputSchema: {
        type: "object",
        properties: {
          database_id: { type: "string" },
          branch: { type: "string" },
          sql: { type: "string", description: "SQL query to execute (SELECT only)" },
          limit: { type: "number", description: "Max rows to return (default: 100)" },
        },
        required: ["database_id", "sql"],
      },
    },
    // ... other tools
  ],
}));
```

**Testing**:
- `Unit: MCP server initializes with correct capabilities`
- `Unit: dbaas_get_schema returns table definitions in text format`
- `Unit: dbaas_execute_query rejects non-SELECT queries`
- `Unit: dbaas_execute_query enforces row limit`
- `Unit: dbaas_create_branch creates branch and returns branch ID`
- `Integration: MCP client can connect and call tools`
- `Unit: authentication via API key is required for all tools`

---

## Phase 11: Usage Metering and Billing

### Purpose

Implement usage metering (compute hours, storage, data transfer) and billing integration. After this phase, the platform tracks per-organization resource consumption and can generate invoices via Stripe.

### Tasks

#### 11.1 — Usage Metering Collector

**What**: Implement background metering that records compute hours, storage consumption, and data transfer per database per billing period.

**Design**:

```typescript
// apps/api/src/workers/metering.worker.ts

// Runs every 5 minutes
// For each active endpoint:
//   1. Record compute_hours = current_cu * (5 / 60) hours
//   2. Record storage_bytes from managed database disk usage
//   3. Record data_transfer from network metrics
//
// Stores in usage_records table (partitioned by period_start)

interface UsageMetrics {
  compute_hours: number;
  storage_bytes: number;
  rows_read: number;
  rows_written: number;
  data_transfer_bytes: number;
  active_endpoints: number;
  branches_count: number;
}

// Usage records are aggregated into the JSONB metrics column:
// INSERT INTO usage_records (organization_id, project_id, database_id, metrics, period_start, period_end)
// VALUES ($1, $2, $3, $4, $5, $6)
// ON CONFLICT (organization_id, project_id, period_start) DO UPDATE
//   SET metrics = usage_records.metrics || $4
```

**Testing**:
- `Unit: metering worker records compute_hours based on current_cu`
- `Unit: idle endpoints (current_cu = 0) record 0 compute_hours`
- `Unit: storage_bytes reflects actual disk usage from managed database`
- `Unit: usage records are partitioned by period_start`
- `Integration: 5-minute metering interval produces correct aggregates over 1 hour`

---

#### 11.2 — Billing API and Stripe Integration

**What**: Implement billing summary endpoints and Stripe subscription management.

**Design**:

```typescript
// GET /v1/organizations/:orgId/billing/summary
interface BillingSummaryResponse {
  organization_id: string;
  period: { start: string; end: string };
  plan: OrgPlan;
  usage: {
    compute_hours: { used: number; unit_price_cents: number; total_cents: number };
    storage_gb: { used: number; unit_price_cents: number; total_cents: number };
    data_transfer_gb: { used: number; unit_price_cents: number; total_cents: number };
  };
  total_cents: number;
  currency: string;
}

// POST /v1/organizations/:orgId/billing/subscribe
interface SubscribeRequest {
  plan: "pro" | "team";
  payment_method_id: string;    // Stripe payment method
  billing_cycle: "monthly" | "annual";
}
```

**Testing**:
- `Unit: billing summary aggregates usage records for current period`
- `Unit: billing summary applies correct unit prices per plan`
- `Unit: subscribe endpoint creates Stripe subscription`
- `Unit: plan upgrade takes effect immediately with prorated billing`
- `Unit: plan downgrade takes effect at end of billing period`
- `Integration (mocked Stripe): full subscribe-bill-invoice cycle`

---

## Phase 12: Multi-Engine Support and Production Hardening

### Purpose

Add MySQL as a second managed engine, implement production security hardening, and prepare for production deployment with Kubernetes Helm charts. After this phase, the platform is production-ready.

### Tasks

#### 12.1 — MySQL Engine Support

**What**: Extend the provisioning service to support MySQL 8.0 instances alongside PostgreSQL.

**Design**:

```typescript
// Engine abstraction:
export interface EngineAdapter {
  provision(config: EngineConfig): Promise<InstanceInfo>;
  createRole(instanceId: string, role: RoleConfig): Promise<void>;
  executeBackup(instanceId: string, target: string): Promise<BackupResult>;
  getMetrics(instanceId: string): Promise<DatabaseMetrics>;
  getSchema(instanceId: string): Promise<SchemaDefinition>;
}

export class PostgreSQLAdapter implements EngineAdapter { /* ... */ }
export class MySQLAdapter implements EngineAdapter { /* ... */ }

export function getEngineAdapter(engine: DatabaseEngine): EngineAdapter {
  switch (engine) {
    case "postgresql": return new PostgreSQLAdapter();
    case "mysql": return new MySQLAdapter();
  }
}
```

MySQL-specific configuration defaults:

```typescript
const DEFAULT_MYSQL_CONFIG: Partial<EngineConfig> = {
  max_connections: 151,
  innodb_buffer_pool_size: "256M",
  innodb_flush_log_at_trx_commit: 1,
  binlog_format: "ROW",
  character_set_server: "utf8mb4",
};
```

**Testing**:
- `Unit: POST /v1/databases with engine "mysql" provisions MySQL instance`
- `Unit: MySQL adapter applies correct default config`
- `Unit: MySQL backup uses mysqldump with --single-transaction`
- `Unit: MySQL metrics collector reads from information_schema and performance_schema`
- `Integration (Docker): MySQL instance provisioned and accessible`

---

#### 12.2 — Production Security Hardening

**What**: Implement security hardening measures: rate limiting, IP allowlisting, API key rotation, and security headers.

**Design**:

```typescript
// Rate limiting per API key
// apps/api/src/middleware/rate-limit.ts

import { FastifyRateLimit } from "@fastify/rate-limit";

// Default: 100 requests per minute per API key
// Authenticated: 1000 requests per minute
// Write operations: 50 per minute

// IP allowlist enforcement per project
// Checked in auth middleware when project settings.ip_allowlist is set

// Security headers (via @fastify/helmet)
// - Strict-Transport-Security: max-age=31536000; includeSubDomains
// - X-Content-Type-Options: nosniff
// - X-Frame-Options: DENY
// - Content-Security-Policy: default-src 'self'

// API key rotation
// POST /v1/api-keys/:id/rotate
// Creates new key, marks old key for deactivation in 24 hours
```

**Testing**:
- `Unit: rate limiter returns 429 after exceeding request limit`
- `Unit: rate limit headers (X-RateLimit-Limit, X-RateLimit-Remaining) are present`
- `Unit: IP allowlist blocks requests from non-allowed IPs`
- `Unit: API key rotation creates new key and schedules old key deactivation`
- `Unit: security headers are present on all responses`
- `Unit: TLS 1.3 is enforced (TLS 1.2 connections rejected)`

---

#### 12.3 — Kubernetes Helm Charts

**What**: Create Helm charts for production deployment of all platform components.

**Design**:

```
helm/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── worker-deployment.yaml
│   ├── console-deployment.yaml
│   ├── console-service.yaml
│   ├── mcp-deployment.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── hpa.yaml                # Horizontal Pod Autoscaler
│   └── pdb.yaml                # PodDisruptionBudget
```

Key `values.yaml` sections:

```yaml
api:
  replicas: 3
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits: { cpu: "2000m", memory: "2Gi" }

worker:
  replicas: 2
  concurrency: 10

console:
  replicas: 2

postgresql:
  external: true
  host: ""
  port: 5432

redis:
  external: true
  host: ""
  port: 6379

ingress:
  enabled: true
  className: nginx
  tls: true
```

**Testing**:
- `Unit: helm template renders valid Kubernetes manifests`
- `Unit: helm lint passes with no errors`
- `Unit: HPA scales API pods based on CPU utilization`
- `Unit: PDB maintains at least 1 replica during rolling updates`
- `Fixture-based: values.yaml with production settings renders correct resource limits`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Core API                     ─── requires Phase 1
    │
Phase 3: Database Provisioning        ─── requires Phase 2
    │
    ├── Phase 4: Branching & CoW      ─── requires Phase 3
    │       │
    │       ├── Phase 5: Backup & PITR    ─── requires Phase 3
    │       │
    │       └── Phase 7: CLI Tool         ─── requires Phase 3, can parallel with 5 & 6
    │
    ├── Phase 6: Management Console       ─── requires Phase 2, can parallel with 4 & 5
    │
    ├── Phase 8: Observability            ─── requires Phase 3, can parallel with 4-7
    │
    └── Phase 9: VCS Integration          ─── requires Phase 4
         │
Phase 10: AI Operations                   ─── requires Phase 3
    │
Phase 11: Billing                          ─── requires Phase 3, can parallel with 9-10
    │
Phase 12: Production Hardening            ─── requires Phases 1-11
```

### Parallelism Opportunities

- Phases 4, 5, 6, 7, 8 can be developed concurrently after Phase 3
- Phase 10 can be developed in parallel with Phase 9
- Phase 11 can be developed in parallel with Phases 9 and 10
- Phase 12 must wait for all other phases

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with working code.
2. All unit tests pass (`pnpm turbo test`).
3. All integration tests pass (with mocked external dependencies where noted).
4. ESLint passes with zero errors (`pnpm turbo lint`).
5. Prettier formatting is consistent (`pnpm turbo format:check`).
6. TypeScript compiler reports zero errors (`pnpm turbo typecheck`).
7. Docker Compose services start and health-check green (`docker compose up -d && docker compose ps`).
8. New API endpoints appear in the auto-generated OpenAPI spec at `/docs`.
9. Database migrations are created and apply cleanly to a fresh database.
10. New environment variables are documented in `.env.example`.
11. Feature works end-to-end via API (and console/CLI where applicable).
12. Audit log captures all mutating operations from the phase.
