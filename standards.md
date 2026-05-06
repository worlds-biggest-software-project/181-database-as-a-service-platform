# Standards & API Reference

> Project: Database as a Service Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 9075:2023 (SQL Standard)**
- URL: https://www.iso.org/standard/76583.html
- The current SQL standard, revised in 2023. Part 1 (Framework), Part 2 (Foundation), Part 3 (Call-Level Interface / SQL/CLI), Part 9 (Management of External Data / SQL/MED), and Part 16 (SQL/PGQ for property graph queries) are all relevant to a DBaaS platform. Any SQL-speaking DBaaS must document its compliance level against SQL:2023, particularly for JSON type support, window functions, and the new property-graph extensions. DBaaS vendors vary significantly in compliance; specifying a conformance target is an architectural decision.

**ISO/IEC 27001:2022 (Information Security Management)**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management systems (ISMS). Enterprise buyers expect DBaaS providers to hold ISO 27001 certification as a baseline. Certification scope must cover data handling, access control, cryptographic key management, incident response, and business continuity — all directly relevant to a managed database service.

**ISO/IEC 27017:2015 (Cloud Security Controls)**
- URL: https://www.iso.org/standard/43757.html
- Extends ISO 27001 with cloud-specific security controls. Relevant to a multi-tenant DBaaS for defining tenant isolation responsibilities, virtual machine hardening, and administrator access governance. Referenced by SOC 2 auditors when assessing cloud database providers.

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational HTTP specification. Any REST management API or Data API exposed by a DBaaS must conform to HTTP semantics for methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation. Replaces RFC 7231.

**RFC 9113 — HTTP/2**
- URL: https://www.rfc-editor.org/rfc/rfc9113
- HTTP/2 framing and multiplexing. DBaaS management APIs and serverless data APIs benefit from HTTP/2 for reduced connection overhead, particularly in high-concurrency serverless invocation patterns.

**RFC 8446 — TLS 1.3**
- URL: https://www.rfc-editor.org/rfc/rfc8446
- Transport Layer Security 1.3. All database connections (wire protocol) and management API traffic must use TLS 1.3 as the minimum version. Required for PCI DSS compliance and HIPAA technical safeguards.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The OAuth 2.0 framework governing how management APIs issue access tokens. The Client Credentials grant (RFC 6749 §4.4) is the standard flow for service-to-service DBaaS API access (machine identity, no user interaction). All major DBaaS management APIs use OAuth 2.0 or API key variants built on similar token patterns.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- JWT is the standard token format used by DBaaS authentication layers (Supabase Auth, Neon Data API, MongoDB Atlas). Row Level Security policies in Postgres-based platforms routinely inspect JWT claims to enforce per-tenant data isolation.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines `Link` headers for pagination in REST APIs. DBaaS management APIs that return lists of databases, branches, or backups should use RFC 8288 link relations for cursor-based pagination.

**RFC 7617 — HTTP Basic Authentication**
- URL: https://www.rfc-editor.org/rfc/rfc7617
- Referenced by some legacy DBaaS management APIs (e.g., MongoDB Atlas legacy API key auth uses HTTP Digest, a related scheme). New platforms should prefer Bearer tokens (RFC 6750) over Basic Auth.

**RFC 6902 — JSON Patch**
- URL: https://www.rfc-editor.org/rfc/rfc6902
- Standard format for partial JSON document updates. Relevant to DBaaS REST APIs that accept configuration updates (e.g., updating cluster settings) without requiring a full resource replacement.

### Data Model & API Specifications

**OpenAPI 3.1 (formerly Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard specification for describing REST APIs. All DBaaS management APIs should publish an OpenAPI 3.1 document to enable SDK code generation, documentation portals, and tooling integration. Neon and MongoDB Atlas both publish OpenAPI specs. Adopting OpenAPI 3.1 (vs 3.0) enables JSON Schema 2020-12 compatibility.

**Apache Arrow ADBC (Arrow Database Connectivity) v1.1.0**
- URL: https://arrow.apache.org/adbc/current/index.html
- A modern, columnar alternative to JDBC and ODBC for analytical workloads. ADBC eliminates row-by-row serialisation overhead by returning Arrow record batches directly. Relevant for a DBaaS offering analytics or HTAP capabilities. dbt Fusion engine and other modern data tools are adopting ADBC as their driver layer.

**JDBC (Java Database Connectivity)**
- URL: https://docs.oracle.com/en/java/jdbc/
- The standard Java API for relational database access. Any SQL-compatible DBaaS must provide or maintain compatibility with a JDBC driver to integrate with Java and JVM-based applications, ORMs (Hibernate, JOOQ), and BI tools.

**ODBC (Open Database Connectivity) — Microsoft / ISO/IEC 9075 Part 3**
- URL: https://learn.microsoft.com/en-us/sql/odbc/reference/odbc-programmer-s-reference
- The C-language database connectivity standard. ODBC drivers are required for integration with Excel, Tableau, Power BI, and other business intelligence tools. Modern cloud data sources (Snowflake, BigQuery) expose ODBC drivers; a DBaaS platform should do the same.

**PostgreSQL Wire Protocol (Frontend/Backend Protocol v3)**
- URL: https://www.postgresql.org/docs/current/protocol.html
- The de-facto network protocol for SQL databases. Many DBaaS platforms (Neon, CockroachDB, AlloyDB, PlanetScale Postgres) implement this protocol to gain access to the entire PostgreSQL driver ecosystem. Supporting the PostgreSQL wire protocol is the fastest path to ecosystem compatibility.

**PostgREST Specification**
- URL: https://postgrest.org/en/stable/
- A widely adopted convention for turning a PostgreSQL schema into a RESTful HTTP API. Supabase and Neon's Data API implement PostgREST compatibility, enabling client libraries from the PostgREST ecosystem to work without modification.

**GraphQL (June 2018 Specification)**
- URL: https://spec.graphql.org/June2018/
- The query language and runtime specification. Several DBaaS platforms (Supabase via pg_graphql, MongoDB Atlas Data API) expose GraphQL endpoints. Useful for client-driven queries where the API consumer shapes the response structure.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- Used for validating API request/response payloads, database document schemas (MongoDB), and OpenAPI 3.1 schema definitions. A DBaaS management API should use JSON Schema to define and validate all resource shapes.

### Security & Authentication Standards

**OAuth 2.0 Client Credentials Grant (RFC 6749 §4.4)**
- URL: https://www.rfc-editor.org/rfc/rfc6749#section-4.4
- The recommended machine-to-machine authentication pattern for DBaaS management APIs. MongoDB Atlas (service accounts), Supabase service role keys, and CockroachDB Cloud API all use variants of this pattern. Access tokens should be short-lived (1 hour) and refreshable.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Relevant for DBaaS platforms that support SSO login for console access (enterprise requirement) and for platforms (like Supabase) that act as identity providers for end-user authentication.

**SCIM 2.0 (RFC 7642/7643/7644)**
- URL: https://www.rfc-editor.org/rfc/rfc7643
- System for Cross-domain Identity Management. Enterprise DBaaS buyers expect SCIM support for automated user provisioning and deprovisioning via their corporate IdP (Okta, Azure AD, etc.). Required for SOC 2 access control auditability.

**OWASP Top 10 for APIs (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The de-facto reference for API security risks. DBaaS management APIs must be designed to mitigate Broken Object Level Authorisation, Broken Authentication, and Excessive Data Exposure — the top risks for multi-tenant API surfaces.

**NIST SP 800-53 Rev 5 (Security and Privacy Controls)**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- U.S. federal security controls framework. FedRAMP authorisation (required to sell to U.S. federal agencies) is based on NIST SP 800-53. Relevant for DBaaS platforms targeting government or regulated-industry customers.

**PCI DSS v4.0 (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/document_library/
- Required for any platform storing or processing payment card data. Mandates TLS 1.2+, encryption at rest, access logging, vulnerability scanning, and annual penetration testing. DBaaS providers must document PCI DSS scope and provide compliance attestation to customers.

**HIPAA Security Rule (45 CFR Part 164)**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- U.S. regulation for protected health information (PHI). DBaaS platforms targeting healthcare must support: audit logging for all data access, encryption in transit and at rest, access controls, and Business Associate Agreements (BAAs). AWS, GCP, and Azure all offer signed BAAs; a competitor DBaaS must do the same.

**GDPR Article 25 (Privacy by Design and by Default)**
- URL: https://gdpr.eu/article-25-data-protection-by-design/
- EU regulation requiring that data protection is built into system design, not added as an afterthought. Relevant for a DBaaS supporting EU customers: data residency controls, right-to-erasure APIs, and data processing agreements are architectural requirements.

### MCP Server Specifications

**Model Context Protocol (MCP) Specification 2025-11-25**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- The open standard (released by Anthropic, now community-governed) defining how AI models connect to external tools and data sources. MCP servers for databases allow AI coding assistants (Claude, Cursor, Windsurf, Cline) to read schemas, execute queries, and manage resources. As of early 2026, MCP database servers exist for PostgreSQL, SQLite, MongoDB, Snowflake, and BigQuery. A DBaaS platform should ship a first-party MCP server to support AI-native developer workflows.

**MCP Toolbox for Databases (Google Cloud)**
- URL: https://cloud.google.com/blog/products/ai-machine-learning/mcp-toolbox-for-databases-now-supports-model-context-protocol
- Google's open-source MCP server supporting AlloyDB, Cloud SQL, Spanner, Bigtable, MySQL, and PostgreSQL. Demonstrates the reference architecture for an enterprise MCP database server, including connection pooling, authentication, and schema introspection tools.

---

## Similar Products — Developer Documentation & APIs

### AWS RDS / Aurora

- **Description:** Amazon's flagship managed relational database service supporting PostgreSQL, MySQL, Oracle, SQL Server, and MariaDB, including Aurora (custom MySQL/PostgreSQL-compatible engine with proprietary storage).
- **API Documentation:** https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/Welcome.html
- **SDKs/Libraries:**
  - Python (boto3): https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds.html
  - JavaScript (AWS SDK v3): https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/RDS.html
  - Java (AWS SDK 2.x): https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/java_rds_code_examples.html
  - Go: https://docs.aws.amazon.com/sdk-for-go/api/service/rds/
- **Developer Guide:** https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/ProgrammingGuide.html
- **Standards:** REST/JSON (query string parameters, not JSON body), IAM SigV4 request signing
- **Authentication:** AWS IAM SigV4 signing; IAM database authentication (token-based, 15-minute TTL); Secrets Manager integration for credential rotation

---

### Supabase

- **Description:** Open-source Postgres development platform providing database, authentication, auto-generated REST and GraphQL APIs, realtime subscriptions, object storage, and edge functions in one integrated product.
- **API Documentation:** https://supabase.com/docs/guides/api
- **SDKs/Libraries:**
  - JavaScript/TypeScript (`supabase-js`): https://supabase.com/docs/reference/javascript/introduction
  - Python (`supabase-py`): https://supabase.com/docs/reference/python/introduction
  - Swift, Dart, C#, Kotlin: https://supabase.com/docs/reference
- **Developer Guide:** https://supabase.com/docs/guides/getting-started
- **Standards:** REST (PostgREST), GraphQL (pg_graphql), PostgreSQL wire protocol, OpenAPI auto-generated from schema
- **Authentication:** API key (anon key / service role key); JWT (RS256) for row-level security; OAuth 2.1 for user auth; SAML 2.0 for enterprise SSO

---

### Neon

- **Description:** Serverless PostgreSQL with disaggregated storage and compute, instant copy-on-write branching, scale-to-zero, and a PostgREST-compatible stateless HTTP Data API.
- **API Documentation (Management):** https://api-docs.neon.tech/reference/getting-started-with-neon-api
- **API Documentation (Data):** https://neon.com/docs/data-api/overview
- **SDKs/Libraries:**
  - TypeScript/JavaScript (`@neondatabase/serverless`): https://neon.com/docs/serverless/serverless-driver
  - TypeScript Data API client (`@neondatabase/neon-js`): https://neon.com/docs/data-api/overview
  - Python SDK (Neon Management API): https://neon.com/docs/reference/python-sdk
- **Developer Guide:** https://neon.com/docs/introduction
- **Standards:** OpenAPI 3.0 (management API), PostgREST (Data API), PostgreSQL wire protocol, REST/JSON
- **Authentication:** Bearer token (API key) for management API; JWT for Data API row-level access; OAuth 2.0 integration for console SSO

---

### PlanetScale

- **Description:** MySQL-compatible serverless database built on Vitess with schema branching, deploy requests for peer-reviewed DDL, and a PostgreSQL product (GA September 2025). Designed for extreme write scale.
- **API Documentation:** https://planetscale.com/docs/api/overview
- **SDKs/Libraries:**
  - JavaScript/Node.js (`@planetscale/database`): https://github.com/planetscale/database-js
  - MySQL-compatible drivers (standard JDBC/ODBC/mysql2 work via wire protocol)
- **Developer Guide:** https://planetscale.com/docs
- **Standards:** REST/JSON (management API); MySQL wire protocol (data plane); OAuth 2.0 for OAuth applications; service token pattern for CI/CD
- **Authentication:** Service tokens for API access; OAuth 2.0 for third-party application integration; standard MySQL credentials for data plane

---

### CockroachDB

- **Description:** Distributed SQL database with PostgreSQL wire compatibility, strong SERIALIZABLE isolation, multi-region active-active writes, and a serverless tier that scales to zero.
- **API Documentation (Cloud):** https://www.cockroachlabs.com/docs/api/cloud/v1.html
- **API Documentation (Cluster):** https://www.cockroachlabs.com/docs/api/cluster/v2.html
- **SDKs/Libraries:**
  - Standard PostgreSQL drivers (psycopg2, pg, pgx) work via wire protocol
  - Terraform Provider: https://registry.terraform.io/providers/cockroachdb/cockroachdb/latest
- **Developer Guide:** https://www.cockroachlabs.com/docs/stable/
- **Standards:** REST/JSON (Cloud API and Cluster API); PostgreSQL wire protocol; Bearer token auth; OpenAPI specification published for Cloud API
- **Authentication:** Cloud API: service account API keys (Bearer token); Cluster API: session tokens via `cockroach auth-session`; data plane: TLS client certificates or password auth

---

### MongoDB Atlas

- **Description:** Fully managed document database with global multi-cloud clusters, integrated full-text search, vector search for AI applications, data federation, triggers, and serverless functions.
- **API Documentation:** https://www.mongodb.com/docs/api/doc/atlas-admin-api-v2/
- **SDKs/Libraries:**
  - Go SDK: https://www.mongodb.com/docs/atlas/sdk/go/
  - Node.js Driver: https://www.mongodb.com/docs/drivers/node/
  - Python (PyMongo): https://www.mongodb.com/docs/drivers/pymongo/
  - Java, C#, PHP, Ruby, Swift, Kotlin: https://www.mongodb.com/docs/drivers/
  - Kafka Connector, Spark Connector: https://www.mongodb.com/products/integrations
- **Developer Guide:** https://www.mongodb.com/docs/atlas/configure-api-access/
- **Standards:** REST/JSON; OpenAPI 3.0 (Admin API v2); MongoDB wire protocol; OAuth 2.0 Client Credentials (service accounts); HTTP Digest (legacy API keys)
- **Authentication:** Service accounts with OAuth 2.0 Client Credentials flow (recommended); legacy API keys using HTTP Digest; OIDC for console SSO; X.509 client certificates for data plane

---

### Aiven

- **Description:** Multi-cloud managed open-source data platform supporting PostgreSQL, MySQL, Kafka, OpenSearch, ClickHouse, Redis/Valkey, Cassandra, and InfluxDB across AWS, GCP, Azure, and two additional cloud providers.
- **API Documentation:** https://api.aiven.io/doc/
- **SDKs/Libraries:**
  - CLI (`avn`): https://aiven.io/docs/tools/cli
  - Terraform Provider: https://registry.terraform.io/providers/aiven/aiven/latest/docs
  - Kubernetes Operator: https://aiven.io/docs/tools/kubernetes
  - Postman Collection: https://aiven.io/developer/get-to-know-the-aiven-api-with-postman
- **Developer Guide:** https://aiven.io/docs/tools/api
- **Standards:** REST/JSON; proprietary `aivenv1 <TOKEN>` Authorization header scheme; all underlying databases expose their native wire protocols
- **Authentication:** Aiven authentication tokens (non-expiring optional, set expiry in hours); token scoped per project or organisation

---

### Turso (libSQL)

- **Description:** Edge-native SQLite-compatible database based on libSQL (Apache-licensed SQLite fork) with unlimited databases for multi-tenant patterns, embedded replicas for offline-first apps, and native vector search.
- **API Documentation:** https://docs.turso.tech/api-reference/introduction
- **SDKs/Libraries:**
  - TypeScript/JavaScript: https://docs.turso.tech/sdk/ts/quickstart
  - Python: https://docs.turso.tech/sdk/python/quickstart
  - Go: https://docs.turso.tech/sdk/go/quickstart
  - Rust: https://docs.turso.tech/sdk/rust/quickstart
  - PHP, Java: https://docs.turso.tech/sdk
- **Developer Guide:** https://docs.turso.tech
- **Standards:** libSQL HTTP API and WebSocket protocol; SQLite wire protocol (embedded replicas); REST/JSON (Platform API); Bearer token auth
- **Authentication:** Turso auth tokens (JWT) for database access; Platform API tokens for management operations; database-level token issuance via Platform API for per-tenant access control

---

## Notes

**Emerging standard worth monitoring — ADBC:** Apache Arrow Database Connectivity (ADBC v1.1.0) is gaining adoption as the columnar successor to JDBC/ODBC for analytical workloads. The dbt Fusion engine adopted it in 2026. A DBaaS platform targeting analytics or HTAP workloads should plan ADBC driver support alongside traditional JDBC/ODBC.

**MCP as the AI integration layer:** The Model Context Protocol (MCP) has rapidly become the default way AI coding assistants access databases in 2026. Shipping a first-party MCP server for a DBaaS platform is now a developer-experience expectation rather than a differentiator.

**PostgreSQL wire protocol as the portability layer:** Implementing the PostgreSQL wire protocol (in addition to or instead of MySQL) provides instant compatibility with the largest driver ecosystem. CockroachDB, Neon, Supabase, PlanetScale Postgres, and Google AlloyDB all share this protocol, enabling standard tooling (psql, pgAdmin, JDBC drivers, ORMs) without custom client SDKs.

**OpenAPI 3.1 vs 3.0:** Most DBaaS management APIs currently publish OpenAPI 3.0 specs. OpenAPI 3.1 (full JSON Schema 2020-12 alignment) is the target standard for new APIs and should be adopted from the outset to avoid migration debt.

**Regulatory evolution:** EU AI Act (effective 2026) and U.S. state privacy laws (CPRA, VCDPA, and others) are expanding the compliance surface. DBaaS platforms handling EU data or AI training workloads should anticipate new data governance and auditability requirements beyond GDPR and HIPAA.
