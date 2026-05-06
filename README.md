# Database-as-a-Service Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native DBaaS that combines self-serve provisioning, instant branching, and scale-to-zero compute without vendor lock-in.

The Database-as-a-Service Platform provides managed database provisioning, backup, monitoring, and access control for engineering teams that want hyperscaler-grade operations without hyperscaler lock-in or pricing complexity. It targets platform/backend engineers, startup CTOs avoiding DBA overhead, and enterprises centralising multi-team database governance.

---

## Why Database-as-a-Service Platform?

- **Hyperscaler lock-in and opaque pricing.** AWS RDS / Aurora, Google Cloud SQL, and Azure SQL bind users to one cloud and present pricing across instance, storage, I/O, and data transfer that is difficult to forecast.
- **Single-engine silos.** Neon, Supabase, and Turso each excel at one engine; Aiven offers many engines but lacks developer-workflow features like branching.
- **Branching without merging.** Existing branching solutions (Neon CoW, PlanetScale deploy requests, Supabase preview branches) still require manual migration scripts to merge schema changes back to main.
- **Cost forecasting is universally weak.** No incumbent provides transparent, predictable cost estimation before a workload runs, leaving SaaS builders guessing at per-tenant economics.
- **AI assistance is shallow.** Console-level AI features (MongoDB Atlas Performance Advisor, Aiven AI Optimizer) are point recommendations rather than end-to-end schema, migration, and cost-optimisation workflows.

---

## Key Features

### Managed Provisioning and Operations

- Managed PostgreSQL provisioning with zero manual server configuration
- Automated backups with configurable retention and point-in-time recovery (PITR)
- Encryption at rest (AES-256) and in transit (TLS 1.3) enforced by default
- Role-based access control with per-project permission scopes
- Read replica provisioning with automatic geo-routing

### Developer Workflow

- Instant copy-on-write database branching for CI/CD integration
- Scale-to-zero compute with automatic resume on first connection
- GitHub and GitLab integrations for automatic branch-per-pull-request lifecycle
- REST management API (OpenAPI 3.0) with API key and OAuth 2.0 authentication
- CLI tool for developer operations

### AI-Assisted Database Operations

- AI-assisted schema and index recommendations from query pattern analysis
- Natural-language query interface in the developer console
- MCP server endpoint for AI assistant integration
- Anomaly detection on query patterns, locking contention, and replication lag

### Observability and Multi-Engine Support

- Observability export: Prometheus metrics, OpenTelemetry traces, structured logs
- Multi-engine support with MySQL compatibility layer as a second engine
- Standard client protocols: JDBC, ODBC, PostgreSQL/MySQL wire protocol

### Multi-Tenant and Compliance (Backlog)

- Database-per-tenant provisioning API for SaaS multi-tenant patterns
- Cross-tenant cost attribution and optimisation dashboard
- Geo-partitioning and data residency controls
- Built-in pgvector-powered vector search as a first-class console feature
- Automated compliance reporting for SOC 2, HIPAA, and GDPR audit requirements

---

## AI-Native Advantage

AI is applied across the database lifecycle rather than as a single console widget. The platform analyses query patterns to propose schema and index changes and generates zero-downtime migration plans automatically. ML models trained on historical workload patterns drive predictive auto-scaling that provisions capacity before demand spikes instead of reacting to thresholds. A natural-language query interface translates plain English to optimised SQL with result explanation, and cross-tenant cost analysis recommends caching, read-replica routing, or query refactoring to minimise per-query spend.

---

## Tech Stack & Deployment

The platform is designed around managed PostgreSQL with a Neon-compatible or self-managed backend, exposing the standard PostgreSQL wire protocol so existing JDBC, ODBC, and psql tooling works unchanged. A REST management API published via OpenAPI 3.0 supports API key and OAuth 2.0 authentication. Relevant standards include SQL:2023, JDBC/ODBC, OpenTelemetry, TLS 1.3, and OAuth 2.0 / SCIM for IAM integration. A multi-engine roadmap adds MySQL compatibility in v1.1.

---

## Market Context

The global cloud database and DBaaS market is valued at approximately $23–35 billion in 2025 and projected to reach $50–91 billion by 2030–2034 at 15–20% CAGR (Grand View Research, IMARC Group). Incumbent pricing spans instance-based hourly billing (AWS RDS), serverless usage-based (Neon, PlanetScale Serverless, CockroachDB Serverless from ~$300/month dedicated), and flat monthly tiers (Supabase Pro $25/month, Turso Developer $4.99/month). Primary buyers are platform/backend engineers, startup CTOs, enterprise database governance teams, and data platform teams.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
