# Database as a Service Platform — Feature & Functionality Survey

> Candidate #181 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| AWS RDS / Aurora | Cloud SaaS | Proprietary / Pay-per-use | https://aws.amazon.com/rds/ |
| Supabase | Open-source / SaaS | Apache 2.0 / Free + paid tiers | https://supabase.com |
| Neon | SaaS | Proprietary / Usage-based | https://neon.com |
| PlanetScale | SaaS | Proprietary / Tiered monthly | https://planetscale.com |
| CockroachDB | Open-source / SaaS | BSL 1.1 / Free tier + enterprise | https://www.cockroachlabs.com |
| MongoDB Atlas | SaaS | Proprietary / Free + dedicated | https://www.mongodb.com/atlas |
| Aiven | SaaS | Proprietary / Per-hour | https://aiven.io |
| Turso (libSQL) | Open-source / SaaS | MIT / Usage-based | https://turso.tech |

---

## Feature Analysis by Solution

### AWS RDS / Aurora

**Core features**
- Managed provisioning for MySQL, PostgreSQL, Oracle, SQL Server, and MariaDB
- Multi-AZ high availability with automatic failover
- Automated backups with configurable retention and point-in-time recovery (PITR)
- Aurora Serverless v2 (now called "Aurora Serverless"): on-demand scaling in 0.5 ACU increments, scales to zero
- Read replica auto-scaling driven by CloudWatch metrics (CPU, connections)
- Performance Insights: real-time monitoring and query-level load analysis
- Amazon DevOps Guru for RDS: ML-powered anomaly detection and diagnostic recommendations
- Storage auto-scaling (no manual resize required)
- Encryption at rest (AES-256) and in transit (TLS)
- VPC isolation with security group controls

**Differentiating features**
- Aurora's custom storage layer gives up to 5× throughput over standard MySQL and delivers up to 30% better performance than the previous platform generation (April 2026 update)
- Global Database feature allows a single Aurora cluster to span multiple AWS regions with sub-second replication
- Aurora I/O-Optimized pricing option for write-heavy workloads eliminates per-I/O charges
- Babelfish compatibility layer lets SQL Server applications run on Aurora PostgreSQL without code changes

**UX patterns**
- Console-driven provisioning wizard with sane defaults
- Parameter groups abstract engine configuration; non-trivial to discover advanced options
- CloudWatch dashboards require manual setup; Performance Insights is a separate console pane
- Complexity is hidden behind instance type selection, which can mislead on actual cost

**Integration points**
- AWS SDK: Java, Python (boto3), Go, JavaScript, Ruby, .NET
- IAM database authentication (token-based, 15-minute expiry)
- AWS Secrets Manager integration for credential rotation
- EventBridge for RDS event notification
- Enhanced Monitoring via CloudWatch Logs
- Supports standard JDBC, ODBC, and PostgreSQL/MySQL wire protocol

**Known gaps**
- No native Git-like branching for development workflows
- Limited built-in natural-language or AI query assistance in the console
- Vendor lock-in: Aurora-specific features do not port to other engines
- Cost transparency is poor; pricing across instance, storage, I/O, and data transfer is difficult to forecast
- No built-in vector search; requires adding pgvector extension manually

**Licence / IP notes**
- Fully proprietary SaaS; no open-source components exposed
- Aurora storage engine and replication protocol are proprietary trade secrets

---

### Supabase

**Core features**
- Dedicated PostgreSQL database per project (full Postgres, not a wrapper)
- Auto-generated REST API (PostgREST) and GraphQL API (pg_graphql extension) from database schema
- Row Level Security (RLS) tightly integrated with both auth and API layers
- Built-in authentication: email/password, magic links, OAuth (30+ providers), phone/SMS, SAML 2.0
- Realtime subscriptions: Postgres change events streamed via WebSockets
- Supabase Storage: S3-compatible object storage with bucket policies
- Edge Functions: Deno-based serverless compute at the edge
- Vector embeddings with pgvector integration for AI/RAG use cases
- Database schema branching (preview environments linked to git branches)
- S3 bucket mounting for 97% faster Edge Function cold starts

**Differentiating features**
- "Firebase alternative" positioning: database + auth + storage + functions in one platform
- Auto-generated, always-up-to-date API docs driven from Postgres schema
- Can turn any Supabase project into an OAuth 2.1 identity provider
- Supabase AI Assistant for query generation and schema design in the console
- Database change replication to external warehouses (Snowflake, BigQuery) via CDC

**UX patterns**
- Console is developer-oriented with SQL editor, table explorer, and logs viewer integrated
- Schema editor provides GUI table management but exposes raw SQL for advanced use
- Branching tied to GitHub integration for CI/CD preview environments
- Progressive disclosure: simple operations require no SQL knowledge; advanced features surface as needed

**Integration points**
- Official SDKs: JavaScript/TypeScript (`supabase-js`), Python (`supabase-py`), Swift, Dart, C#, Kotlin
- REST API (PostgREST): standard HTTP, API key auth, JWT auth
- GraphQL endpoint via pg_graphql
- Webhooks via Supabase Realtime and Database Webhooks
- Postgres wire protocol for direct DB connections
- Terraform provider and GitHub Actions integration

**Known gaps**
- Limited to PostgreSQL; no multi-engine support
- Branching is preview-only; merging schema changes requires manual migration scripts
- Edge Functions require Deno runtime (limited Node.js ecosystem compatibility)
- Storage lacks advanced CDN configuration relative to dedicated object stores
- Free tier compute pauses after inactivity

**Licence / IP notes**
- Core Supabase platform is Apache 2.0; self-hostable
- Some enterprise features (SSO, advanced audit logs) are closed-source additions
- PostgREST: MIT; pg_graphql: Apache 2.0

---

### Neon

**Core features**
- Serverless PostgreSQL with disaggregated compute and storage architecture
- Instant copy-on-write (CoW) database branching: a branch from a 50 GB database is ready in under one second
- Scale-to-zero: compute shuts down during inactivity and resumes on first connection
- Point-in-time restore to any timestamp within the retention window
- Neon Data API: stateless HTTP interface (PostgREST-compatible) for web/serverless/edge runtimes
- Built-in read replicas for read-heavy workloads
- Autoscaling compute: capacity adjusts based on load without manual intervention
- IP Allow-listing and branch-level access controls

**Differentiating features**
- CoW branching is the most mature instant-data branching in the market; Neon branches include actual data, unlike PlanetScale MySQL branches (schema-only)
- Claimable Postgres REST API: provision a database for a user programmatically via API call, useful for multi-tenant SaaS builders
- Neon Serverless Driver: WebSocket-based PostgreSQL driver optimised for serverless runtimes where persistent connections are impractical
- Native GitHub integration: automatically creates a Neon branch per pull request, tears it down on merge

**UX patterns**
- Dashboard focused on branch topology: projects, branches, and compute endpoints are first-class objects
- CLI and GitHub Actions as primary developer interfaces; console is secondary
- Pricing shown in compute-hour and storage-hour units; usage dashboards updated in near-real-time

**Integration points**
- Official SDKs: TypeScript/JavaScript (`@neondatabase/serverless`), Python SDK (Neon API)
- Neon API: OpenAPI 3.0 specification; REST; API key auth via Bearer token
- Data API: HTTP interface compatible with PostgREST clients (`@neondatabase/neon-js`, `@neondatabase/postgrest-js`)
- Supports standard PostgreSQL wire protocol (JDBC, ODBC, psql)
- Vercel, Netlify, and Cloudflare Workers integrations (first-class)
- Terraform provider available

**Known gaps**
- PostgreSQL only; no multi-engine support
- Branching merge/promotion still requires manual migration workflow
- Global replica footprint is more limited than hyperscalers
- Connection pooling (via PgBouncer) is managed but the connection limit per branch may constrain very bursty workloads

**Licence / IP notes**
- Core storage engine (Zenith) is open-source (Apache 2.0) on GitHub; SaaS layer is proprietary
- No patent concerns identified

---

### PlanetScale

**Core features**
- MySQL-compatible serverless database built on Vitess (Google's YouTube sharding layer)
- PostgreSQL product (GA since September 2025) with its own branching model
- Schema branching and non-blocking online DDL: schema changes never lock tables
- Deploy requests: peer-review and approval workflow for schema changes analogous to GitHub pull requests
- Automatic horizontal sharding via Vitess
- Connection pooling and query caching built into the proxy layer
- Global read replicas for low-latency reads
- Automated backups with PITR

**Differentiating features**
- Deploy requests bring software-engineering review discipline to database schema changes
- Vitess underpins the platform, making extreme write scale (billions of rows, millions of QPS) achievable without application changes
- MySQL branches (schema-only) enable rapid developer iteration without full data copies — trade-off vs Neon's data-inclusive CoW branches
- PlanetScale Boost (query caching layer) transparently caches read queries at the proxy

**UX patterns**
- Web console surfaces branch status and deploy request pipeline prominently
- CLI (`pscale`) is the primary developer workflow tool
- Schema change workflow modelled on pull requests: familiar to developers, foreign to DBAs
- Pricing based on rows read/written per month, which can be counter-intuitive for query-heavy apps

**Integration points**
- Official SDK: `@planetscale/database` (JavaScript/Node.js, Fetch API-compatible)
- Management API: REST; Service token auth or OAuth 2.0
- MySQL wire protocol compatible (standard JDBC/ODBC drivers work)
- Datadog, Terraform, and GitHub Actions integrations
- Prisma, Drizzle, and other ORMs supported natively

**Known gaps**
- MySQL enforces no foreign keys in Vitess mode (referential integrity must be handled in application)
- No built-in vector search
- PostgreSQL product is newer and has fewer features than the MySQL offering
- Data-inclusive branching requires backup restoration (slower than Neon's CoW)

**Licence / IP notes**
- Proprietary SaaS; Vitess itself is Apache 2.0 (open-source from CNCF)
- PlanetScale Boost query caching mechanism is proprietary

---

### CockroachDB

**Core features**
- Distributed SQL with strong SERIALIZABLE isolation and multi-region active-active writes
- PostgreSQL wire protocol compatible (most Postgres drivers and ORMs work out of the box)
- Serverless tier: sub-second scaling from zero, usage-based billing
- Multi-region tables: rows pinned to specific regions for data residency compliance
- Geo-partitioning: query routing and data placement by geography
- Automated failover: survives zone and region failures without manual intervention
- Built-in change data capture (CDC) via changefeeds to Kafka, cloud storage
- HTAP capabilities: analytical queries run alongside transactional workloads

**Differentiating features**
- Only DBaaS with multi-region active-active writes on serverless tier (free tier available)
- Geo-partitioning for regulatory data residency (GDPR, CCPA locality requirements)
- Survives loss of an entire AWS/GCP region with zero data loss (RPO=0, RTO<30s typical)
- CockroachDB Serverless can save up to 80% over dedicated clusters for intermittent workloads

**UX patterns**
- Console includes a visual cluster map showing data distribution across regions
- Serverless and dedicated share the same console UI; switching is a configuration change
- `cockroach sql` CLI for interactive access; standard psql also works
- Geo-partitioning configuration is SQL-based but requires understanding of zone configs

**Integration points**
- Cloud API (REST, Bearer token auth): manage clusters, users, backups programmatically
- Cluster API v2 (REST, session token auth): per-cluster node management
- AWS, GCP, Azure cloud APIs for multi-cloud deployments
- Standard JDBC, ODBC, PostgreSQL wire protocol
- Kafka for CDC changefeeds; S3 / GCS / Azure Blob for backup targets
- Terraform provider; Kubernetes Operator available

**Known gaps**
- Higher latency than single-node databases for simple point-lookups (consensus overhead)
- No native database branching workflow for developers
- Limited vector search support compared to pgvector-enabled alternatives
- Enterprise geo-partitioning features require paid tier

**Licence / IP notes**
- CockroachDB source is Business Source Licence 1.1 (BSL 1.1): free for non-production use; converts to Apache 2.0 after 4 years
- BSL restricts competitors offering CockroachDB as a managed service without a commercial licence

---

### MongoDB Atlas

**Core features**
- Managed document database (BSON/JSON) with flexible schema
- Global clusters: data distributed across AWS, GCP, Azure in 120+ regions
- Atlas Search: full-text search powered by Lucene, embedded in the cluster
- Atlas Vector Search: approximate nearest neighbour (ANN) and exact nearest neighbour (ENN) for AI/RAG workloads; supports Flat Indexes for multi-tenant efficiency
- Atlas Data Federation: query across Atlas, S3, Atlas Data Lake in one aggregation pipeline
- Time series collections: purpose-optimised for IoT and telemetry data
- Atlas Triggers and Atlas Functions: serverless event-driven compute
- Online Archive: automatically tier cold data to S3 while keeping it queryable
- Automated backups, PITR, and cross-region snapshot replication

**Differentiating features**
- Unified operational + vector + search platform: no need to sync data between separate stores
- Multitenant vector search with Flat Indexes is simpler and more efficient for per-tenant isolation
- Atlas Data Federation enables cross-cloud, cross-store federated queries from a single endpoint
- Queryable Encryption: client-side field-level encryption with queries possible on encrypted data

**UX patterns**
- Atlas console is feature-rich with dedicated UIs for clusters, search indexes, data federation, charts, and triggers
- Visual schema explorer and document viewer for schema-free exploration
- Performance Advisor and Index Suggestions are AI-assisted recommendations surfaced automatically
- Aggregation Pipeline Builder provides a GUI for constructing complex pipelines

**Integration points**
- Atlas Administration API v2: REST, OAuth 2.0 (Client Credentials flow) for service accounts; legacy HTTP Digest for API keys
- Atlas Go SDK; Node.js, Python, Java, C#, PHP, Ruby, Swift, Kotlin drivers
- Atlas App Services for mobile sync (Device Sync) and serverless functions
- Kafka Connector for MongoDB, Spark Connector, BI Connector (ODBC/JDBC for SQL tools)
- Terraform provider; Kubernetes Operator
- OpenAPI 3.0 specification published for Atlas Admin API v2

**Known gaps**
- Document model mismatch with relational data (joins are expensive aggregation pipeline operations)
- Cost at scale is high; storage and compute are bundled making optimisation difficult
- Queryable Encryption adds application complexity and is not broadly adopted
- Free shared tier ("M0") has strict resource limits unsuitable for production

**Licence / IP notes**
- MongoDB server is Server Side Public Licence (SSPL): use as a cloud service requires commercial licence (hence the Atlas SaaS model)
- Atlas driver libraries are Apache 2.0

---

### Aiven

**Core features**
- Managed open-source databases: PostgreSQL, MySQL, Redis (Valkey), Kafka, OpenSearch, ClickHouse, Cassandra, InfluxDB
- Deployment across 5 major clouds (AWS, GCP, Azure, DigitalOcean, UpCloud) and 100+ regions
- Centralized management console for all services across all clouds
- Built-in observability: metrics to Prometheus, logs via rsyslog, dashboards via integrated Grafana
- Aiven for Metrics: advanced telemetry storage for cross-service observability
- Network peering and VPC integration across clouds
- Encryption at rest and in transit enforced by default
- Automated backups with configurable retention

**Differentiating features**
- Broadest engine portfolio in one platform: transactional, streaming, search, and analytics engines managed uniformly
- AI Database Optimizer: analyses PostgreSQL and MySQL queries, suggests index changes and SQL rewrites with real-time performance insights
- Multi-cloud architecture support from a single control plane (true cloud-agnostic managed service)
- Aiven Terraform Provider and Operator for Kubernetes for IaC-first operations

**UX patterns**
- Console organizes services by project and cloud region; service topology is visible at a glance
- Service cards expose health, connection details, and metrics without navigating away
- No branching or developer workflow features — targeted at ops teams rather than individual developers

**Integration points**
- Aiven REST API: token-based authentication (`aivenv1 <TOKEN>`); comprehensive resource management
- Terraform provider for all Aiven services
- Kubernetes Operator for service provisioning in cluster manifests
- Datadog, Prometheus, and Grafana integrations for observability
- Kafka Connect ecosystem for streaming integrations
- Service integration API for cross-service data flow (e.g., PostgreSQL metrics → InfluxDB)

**Known gaps**
- No database branching or development workflow features
- Less tightly integrated with individual cloud providers than native hyperscaler offerings
- No built-in vector search beyond what PostgreSQL/pgvector provides
- Pricing per-node makes it less competitive for very low-traffic or development workloads

**Licence / IP notes**
- Aiven platform is proprietary; all underlying databases are open-source with their own licences
- PostgreSQL: PostgreSQL Licence; Kafka: Apache 2.0; Redis/Valkey: BSD-3-Clause

---

### Turso (libSQL)

**Core features**
- Edge-native SQLite-compatible database based on libSQL (Apache-licensed fork of SQLite)
- Unlimited databases per account: purpose-built for database-per-tenant multi-tenant architectures
- Embedded replicas: local read-only SQLite file synchronised with the cloud primary
- Data Edge: global replication to edge locations for sub-millisecond read latency
- 4× write throughput over standard SQLite via async I/O improvements in libSQL
- Native vector search via libSQL vector extension
- WASM and browser support for client-side SQLite operations
- Built-in concurrent write support (not available in stock SQLite)

**Differentiating features**
- Database-per-tenant at negligible marginal cost: complete isolation, no noisy-neighbour risk, simplifies compliance
- Offline-first capability via embedded replicas: app works without network, syncs when connected
- Rust-based libSQL rewrite enables async and concurrent operations that SQLite cannot do

**UX patterns**
- Developer plan at $4.99/month includes unlimited databases — removes friction for multi-tenant provisioning
- CLI (`turso`) and API are the primary developer interfaces
- Dashboard shows per-database usage and replica topology
- Simple for read-heavy, edge-located use cases; less discoverable for complex relational workloads

**Integration points**
- libSQL HTTP API and WebSocket protocol for edge runtimes
- Official SDKs: JavaScript/TypeScript, Python, Go, Rust, PHP, Java
- Embedded replica libraries for local-first applications
- Cloudflare Workers, Vercel Edge Functions, Deno Deploy compatible
- Turso Platform API (REST, token auth) for programmatic database provisioning

**Known gaps**
- SQLite compatibility limits complex relational features (limited ALTER TABLE, no stored procedures)
- Global consistency guarantees at write scale are weaker than distributed SQL alternatives
- Not suited for write-heavy, globally distributed workloads requiring strong consistency
- Smaller ecosystem and community compared to PostgreSQL-based alternatives

**Licence / IP notes**
- libSQL is Apache 2.0 (open-source fork of SQLite)
- Turso platform SaaS layer is proprietary
- SQLite itself is public domain; libSQL changes are Apache 2.0

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Automated provisioning with zero manual server configuration
- Automated backups with configurable retention and PITR
- Encryption at rest (AES-256) and in transit (TLS 1.3)
- Connection pooling to handle serverless/edge client patterns
- Horizontal read scaling via read replicas
- Monitoring and alerting integrated with standard observability stacks (Prometheus, Datadog, CloudWatch)
- Role-based access control (RBAC) and least-privilege IAM policies
- Standard client protocol support: JDBC, ODBC, PostgreSQL or MySQL wire protocol
- REST management API with API key or OAuth authentication
- CLI tool for developer operations

### Differentiating Features
- Instant database branching for CI/CD and development workflow (Neon CoW, Supabase preview branches)
- Schema deploy requests / peer-review workflow for schema changes (PlanetScale)
- Scale-to-zero serverless compute that resumes on first connection (Neon, Aurora Serverless)
- Database-per-tenant provisioning at scale with negligible marginal cost (Turso)
- Multi-region active-active writes with geo-partitioning (CockroachDB)
- AI-assisted query optimisation and index recommendations surfaced in-console (MongoDB Atlas, Aiven AI Optimizer)
- Unified platform spanning transactional, search, streaming, and vector workloads (MongoDB Atlas, Aiven)
- Edge replication for sub-millisecond local reads (Turso embedded replicas)
- Queryable Encryption (MongoDB Atlas)

### Underserved Areas / Opportunities
- Unified multi-engine platform with developer-friendly branching (Aiven has engines but no branching; Neon/Supabase have branching but one engine)
- Transparent, predictable cost estimation before a workload runs — all platforms make forecasting difficult
- Natural-language query interface in the console for non-technical operators
- Automated schema migration planning from AI analysis of query logs
- Cross-tenant cost attribution and optimisation dashboard for SaaS builders
- Built-in compliance reporting (SOC 2, GDPR, HIPAA audit logs) without requiring enterprise tier upgrades
- Branching with automatic data masking for PII-safe development environments
- Platform-agnostic migration tooling to move between DBaaS providers without vendor lock-in

### AI-Augmentation Candidates
- Schema design assistant: AI analyses query patterns and proposes optimal schema structure and indexes
- Migration plan generation: AI generates zero-downtime migration scripts from diff analysis
- Predictive auto-scaling: ML-driven capacity planning before demand spikes rather than reactive threshold alerts
- Anomaly detection and self-healing: AI identifies locking contention, replication lag, and unusual query patterns with automated remediation
- Natural-language query translation: convert plain-English queries to optimised SQL with result explanation
- Cost optimisation advisor: cross-tenant analysis of query costs with recommendations for caching, read-replica routing, or query refactoring
- Branch lifecycle management: AI recommends branch pruning and merge strategies based on activity

---

## Legal & IP Summary

No copyright, patent, or trade secret concerns were identified that would prevent building an open-source DBaaS platform. The major open-source engines (PostgreSQL, MySQL, SQLite/libSQL) carry permissive licences (PostgreSQL Licence, GPL v2, Apache 2.0, MIT). CockroachDB's BSL 1.1 restricts competing managed services using CockroachDB source code, but building on top of PostgreSQL or other permissively licenced engines is unencumbered. MongoDB's SSPL presents licence risk if using MongoDB server code directly; using compatible drivers (Apache 2.0) to connect to Atlas is unrestricted. Vitess (PlanetScale's underpinning) is Apache 2.0 via CNCF. No active patent claims were identified in search results for branching, scale-to-zero, or copy-on-write storage techniques as applied to databases, though Snowflake and others have filed storage-layer patents in adjacent spaces that warrant legal review before replicating specific implementations.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Managed PostgreSQL provisioning (via Neon-compatible or self-managed backend)
- REST management API (OpenAPI 3.0 spec) with API key and OAuth 2.0 authentication
- Automated backups with PITR and configurable retention
- Instant database branching (copy-on-write) for CI/CD integration
- Scale-to-zero compute with automatic resume on connection
- Encryption at rest (AES-256) and in transit (TLS 1.3) enforced by default
- Role-based access control with per-project permission scopes

**Should-have (v1.1)**
- AI-assisted schema and index recommendations from query pattern analysis
- Natural-language query interface in the developer console
- Read replica provisioning with automatic geo-routing
- GitHub and GitLab integrations for automatic branch-per-pull-request lifecycle
- Observability export: Prometheus metrics, OpenTelemetry traces, structured logs
- Multi-engine support (MySQL compatibility layer as second engine)
- MCP server endpoint for AI assistant integration

**Nice-to-have (backlog)**
- Database-per-tenant provisioning API for SaaS multi-tenant patterns (Turso-style)
- Cross-tenant cost attribution and optimisation dashboard
- Geo-partitioning and data residency controls for regulatory compliance
- Built-in vector search (pgvector-powered) as a first-class console feature
- Queryable encrypted columns for sensitive-field protection
- Platform-agnostic migration export tool (pg_dump-compatible + metadata)
- Automated compliance reporting for SOC 2, HIPAA, GDPR audit requirements
