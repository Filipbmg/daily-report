**Status:** Architecture / development plan  
**Updated:** 2026-09-16  
**Scope:** Browser-based platform for lawful monitoring and analysis of publicly accessible online material, with strong security, privacy, provenance and analyst controls.

> Collection should follow applicable law, source terms, authorization boundaries, rate limits and internal source policies. The platform should not make credential acquisition, CAPTCHA breaking, abusive anti-bot bypass, or unauthorized access part of the design.

---

## 1. Executive recommendation

Build the first production-capable version as a **modular monolith plus independent worker processes**, not as a large microservice estate.

Recommended core stack:

| Layer | Primary choice | Why |
|---|---|---|
| Browser UI | **TypeScript + React + Next.js** | Strong typing, mature web ecosystem, excellent coding-assistant support |
| API / application backend | **Python + FastAPI** | Strong fit with collection, AI and data-processing code |
| Crawling | **Python + Scrapy** | Mature crawling engine with concurrency, throttling, retries and pipelines |
| JS-heavy pages | **Playwright for Python** | Browser rendering fallback for dynamic content |
| HTML parsing | **Parsel/lxml**, optionally **selectolax** | Reliable selectors and fast parsing |
| Main database | **PostgreSQL** | Transactions, JSONB, mature tooling and row-level security |
| Raw evidence | **S3-compatible object storage** | Suitable for HTML, WARC, screenshots, files and media |
| Search | **PostgreSQL FTS + pgvector initially**, **OpenSearch later** | Avoid a dedicated search cluster until justified |
| Queue/workflows | **Redis + Dramatiq/Celery initially**, **Temporal later** | Simple start with a path to durable workflows |
| Auth | **OIDC/SAML SSO** | Central MFA and access lifecycle |
| Containers | **Docker** | Reproducible dev, CI and deployment environments |
| Local orchestration | **Docker Compose** | Enough for development without Kubernetes overhead |
| Infrastructure | **Terraform/OpenTofu** | Reviewable infrastructure as code |
| CI/CD | **GitHub Actions** | Natural fit for the repo and security tooling |
| Observability | **OpenTelemetry + Prometheus/Grafana or managed equivalent** | Vendor-neutral metrics, traces and logs |

The central domain object should be a **Signal**, not a webpage. A page, post, comment, RSS item, API object or social item becomes a normalized Signal. This keeps downstream analysis independent of collection method.

```text
Sources
  |
  v
Collectors / crawlers
  |
  v
Raw capture + provenance
  |
  v
Normalization
  |
  v
Signal
  |
  +----> Search/indexing
  +----> Entity extraction
  +----> Embeddings/similarity
  +----> Narrative/topic clustering
  +----> Rules/classifiers
  +----> Analyst annotations
  |
  v
Fusion / correlation
  |
  +----> Cases
  +----> Alerts
  +----> Graph/network view
  +----> Reports/exports
```

The key pattern is modular collection feeding a common internal model, with independent enrichment and a separate correlation layer joining actors, content, narratives, entities and events.

---

## 2. Product capabilities

### 2.1 Source management

Each source should be a first-class object with:

- canonical domain/platform
- collection method: HTTP, RSS, API, browser or manual import
- allowed and denied URL patterns
- crawl interval and priority
- per-source concurrency and request rate
- robots and source-policy handling
- expected language/region
- parser/extractor version
- owner/team
- health status and last successful capture

Treat source configuration as data rather than hardcoding every source into spiders.

### 2.2 Collection

Support:

- scheduled crawling
- analyst-triggered collection from an allowed URL
- RSS/Atom ingestion
- public APIs where available
- normal HTTP collection for static pages
- browser rendering only when necessary
- pagination and link-follow rules
- incremental recrawling
- ETag/Last-Modified handling
- retry, exponential backoff and source-specific rate limiting
- deduplication by canonical URL and content hash
- redirect-chain recording
- selected response-header capture
- screenshots or PDF snapshots when justified
- WARC storage where replay/evidentiary retention matters

### 2.3 Canonical Signal model

A Signal should include roughly:

```text
signal_id
source_id
source_type
canonical_url
observed_at
captured_at
published_at
author/account reference
raw_capture reference
content_hash
language
text/title
media references
outbound links
parent/reply/repost relationship
entities
locations if legitimately available
topics/tags
machine analyses
analyst annotations
provenance metadata
case memberships
visibility/access policy
```

Machine-derived analysis should be versioned with:

```text
analysis_type
model/provider
model_version
prompt/pipeline version
input hash
output
confidence/calibration metadata
created_at
```

Never silently overwrite old machine analysis when models change.

### 2.4 Search

Analysts should be able to search by:

- exact words and phrases
- Boolean operators
- domain/source
- author/account
- date/time ranges
- language
- entity
- tag/topic
- case
- URL/domain relationships
- semantic similarity
- similar document/post

Start with PostgreSQL full-text search plus pgvector. Add OpenSearch when richer ranking, faceting, hybrid search, high query concurrency or larger index sizes justify it.

### 2.5 AI-assisted analysis

Useful modules include:

- language detection
- translation
- named-entity recognition
- topic classification
- analyst-defined relevance classification
- embeddings
- near-duplicate and semantic-similarity detection
- narrative clustering
- claim extraction
- summarization
- cross-source comparison
- change detection between captures
- suggested tags
- alert prioritization

AI output should be treated as analyst decision support. For sensitive labels, show the underlying evidence and preserve human verification.

### 2.6 Fusion and correlation

Correlate Signals into higher-order objects:

- entity
- account/actor
- domain
- narrative/claim
- event
- cluster/campaign
- case/investigation

Relationships can include:

```text
ACCOUNT -> POSTED -> SIGNAL
SIGNAL -> LINKS_TO -> DOMAIN
SIGNAL -> MENTIONS -> ENTITY
SIGNAL -> SIMILAR_TO -> SIGNAL
ACCOUNT -> REPLIED_TO -> ACCOUNT
NARRATIVE -> CONTAINS -> SIGNAL
CASE -> CONTAINS -> SIGNAL
```

Use PostgreSQL initially. Add a graph database only when graph traversal becomes a major workload rather than merely a visualization feature.

### 2.7 Cases and analyst workflow

Cases should support:

- case membership and need-to-know access
- saved searches
- watchlists
- notes
- evidence pinning
- timeline view
- entity list
- relationship graph
- analyst tags and confidence
- task/review state
- report/export generation
- immutable audit trail of changes

### 2.8 Alerting

Examples:

- new content matching a saved query
- sudden increase in volume around a narrative or entity
- watched source/account becomes active
- new cross-source appearance of an existing narrative
- high-similarity content across multiple sources
- collection source repeatedly fails

Every alert should explain why it fired and link to the underlying evidence.

### 2.9 Provenance and evidence

For every capture, retain enough information to answer what the system saw, when it saw it and how it was collected:

- requested URL
- final URL and redirect chain
- UTC capture time
- HTTP status
- relevant response headers
- raw response/body reference
- SHA-256 hash
- crawler version/commit SHA
- parser version
- source configuration version
- screenshot if applicable
- WARC reference where enabled

Raw captures should be immutable to normal users. Corrections belong in derived metadata, not by rewriting original evidence.

---

## 3. Collection technology

### Primary: Scrapy

Use Scrapy as the bulk crawling engine for normal HTML sites, discovery, pagination, retry/backoff, throttling, scheduling, item pipelines and resumable crawls.

### Browser fallback: Playwright

Use Playwright only when HTTP collection is insufficient, such as JavaScript-rendered content, client-side pagination or when a screenshot is required.

Do not browser-render every page by default. Browser workers consume much more CPU/RAM, are slower and have a larger attack surface.

### Supporting Python libraries

Recommended:

- `parsel` / `lxml`
- `selectolax` when parsing speed matters
- `httpx`
- `trafilatura` or `readability-lxml`
- `feedparser`
- `tldextract`
- `warcio`
- `pydantic`

### Backup options

**Crawlee:** good if you want collection closer to the TypeScript ecosystem.

**Beautiful Soup + httpx:** excellent for small targeted collectors, but not enough by itself for a substantial crawl platform.

**Selenium:** viable, but Playwright is the cleaner default for new browser automation.

Design for graceful failure rather than evasion. Prefer official APIs/feeds where available, cache aggressively, use conditional requests, and respect source-specific policies and rate limits.

---

## 4. Programming languages and coding assistants

There is no permanent universal ranking of the best language for coding assistants. Repository context, tests, type systems and ecosystem maturity matter at least as much as raw language familiarity.

### Recommended: TypeScript + Python

Use **TypeScript** for the browser UI and shared web contracts. It has a very large modern code corpus, excellent tooling and strong static feedback.

Use **Python** for crawling, browser workers, FastAPI, NLP, embeddings, data pipelines and analyst automation. It has the strongest combined ecosystem for scraping and AI work.

The practical development loop should always be:

```text
spec -> small change -> formatter -> linter -> type checker -> tests -> security checks -> review
```

### Backup: Go

Good for future high-throughput network services, gateways or isolated ingestion daemons where you need small binaries, predictable concurrency and low memory usage.

### Backup: Rust

Good for security-sensitive parsers, sandbox helpers or genuinely performance-critical components. Use surgically rather than as the default application language.

### Backup: C# or Java

Good enterprise choices if the organization already standardizes on those ecosystems. They offer less advantage for a green-field scraping/AI-heavy product than Python plus TypeScript.

---

## 5. Backend and data architecture

### API

Use FastAPI with Pydantic models. Keep the API mostly stateless and responsible for:

- auth/session integration
- authorization
- cases
- signals and metadata
- search
- source configuration
- analyst annotations
- job requests/status
- export requests

Do not let the API fetch arbitrary external URLs directly. Collection requests should go to an isolated worker tier.

### PostgreSQL

Use PostgreSQL as the system of record for users/team mappings, sources, crawl jobs, Signals, entities, relationships, cases, tags, notes, alert definitions, audit metadata and AI analysis records.

Use Row-Level Security as defense in depth when access differs by team, case or tenant.

### Object storage

Store raw HTML, WARC, screenshots, PDFs, images/video, downloaded files and generated reports in object storage referenced by stable IDs and hashes.

### Queue/workflows

Start with Redis plus Dramatiq or Celery. Consider Temporal when workflows become long-lived or multi-step, such as:

```text
crawl -> render fallback -> malware scan -> extract -> translate -> embed -> classify -> index -> alert
```

Do not add Kafka until replay, throughput or many independent consumers clearly justify it.

---

## 6. Security and privacy architecture

Assume every remote page and downloaded file is hostile.

### 6.1 Network isolation

Crawler/browser workers should run in a separate security zone with:

- no route to internal corporate networks
- no direct route to database subnets where avoidable
- cloud metadata endpoints blocked
- loopback/private/link-local/reserved destinations blocked
- IPs rechecked after redirects
- controlled egress proxy/firewall where practical
- DNS logging and policy enforcement

### 6.2 Browser isolation

Browser jobs should be disposable:

- non-root user
- read-only root filesystem where practical
- minimal writable temp space
- CPU/memory/time limits
- seccomp/AppArmor/SELinux where supported
- no host filesystem mounts
- no Docker socket
- no application secrets
- browser context destroyed after collection

### 6.3 Analyst rendering

Never inject collected HTML directly into the authenticated application.

Prefer:

- extracted text
- sanitized HTML only where needed
- proxied images rather than analyst browsers loading remote URLs
- stripped scripts, event handlers, iframes and active content
- strict Content Security Policy
- screenshots for exact visual review

### 6.4 Downloaded files

Use a quarantine pipeline:

```text
collector
  -> quarantine object store
  -> file type validation
  -> malware scanner
  -> metadata extraction in sandbox
  -> approved object state
```

### 6.5 Identity and access

Require:

- enterprise SSO via OIDC/SAML
- MFA, ideally WebAuthn for sensitive deployments
- short-lived sessions
- RBAC for standard roles
- optional ABAC/need-to-know controls for cases
- explicit export permission
- separated admin and analyst roles
- fast revocation
- heavily audited break-glass access

### 6.6 Audit

Audit authentication, permission changes, source configuration changes, case membership, access to restricted cases, evidence export/download, deletion/retention actions and administrative actions.

Prefer append-only/tamper-evident audit storage separated from normal application logs.

### 6.7 Encryption and secrets

- TLS everywhere
- encryption at rest
- KMS-backed keys where available
- no production secrets in Git
- no committed `.env` files
- credential rotation
- separate secret stores per environment

### 6.8 Privacy

Build in purpose classification, retention policy, automatic expiry, legal hold where applicable, deletion workflows, export restrictions and data minimization.

### 6.9 AI privacy

Before external model use, establish whether the data is allowed to leave the environment. For high-sensitivity deployments, privately hosted models may be preferable.

Record model/provider, model version, prompt/pipeline version, input references/hashes, output and timestamp.

---

## 7. Repository design

Use a **monorepo** initially:

```text
/
├── apps/
│   └── web/
├── services/
│   ├── api/
│   ├── crawler/
│   ├── browser-worker/
│   ├── enricher/
│   └── scheduler/
├── packages/
│   └── contracts/
├── migrations/
├── infra/
│   ├── terraform/
│   └── policies/
├── deploy/
│   ├── compose/
│   └── containers/
├── tests/
│   ├── fixtures/
│   ├── integration/
│   └── security/
├── docs/
│   ├── adr/
│   ├── threat-model/
│   ├── data-model/
│   └── runbooks/
├── .github/
│   ├── workflows/
│   ├── CODEOWNERS
│   └── dependabot.yml
├── .env.example
├── pyproject.toml
├── package.json
├── pnpm-lock.yaml
└── README.md
```

A monorepo is especially helpful while UI, API, collectors and shared schemas change together.

---

## 8. Git workflow during development

Use **trunk-based development with short-lived feature branches**.

```text
main
  |
  +-- feature/source-config
  +-- feature/crawler-health
  +-- fix/redirect-validation
```

Rules:

- protect `main`
- no routine direct pushes to `main`
- every code change through pull request
- keep branches short-lived
- prefer squash merge
- delete merged branches
- avoid long-running `develop` branches unless release needs later justify them

A PR should explain:

- why the change exists
- security/privacy impact
- data-model impact
- migration notes
- screenshots for UI changes
- tests added/updated
- rollback/operational notes where relevant

Example commit style:

```text
feat(crawler): add source-specific rate policy
fix(api): reject private redirect destinations
security(browser): remove worker access to metadata network
chore(deps): update playwright
```

### Architecture Decision Records

For expensive-to-reverse decisions, add ADRs in `docs/adr/` describing context, decision, alternatives and consequences.

---

## 9. Local development

Use Docker Compose for infrastructure dependencies while allowing application services to run either locally or in containers.

Typical local services:

```text
postgres
redis
minio
mail/test notification sink
optional opensearch profile
otel collector
```

Goals:

- one documented bootstrap path after clone
- reproducible dependency versions
- synthetic seed data
- no production credentials
- no automatic laptop access to production databases

Use `.env.example` with fake defaults. Real local secrets belong in ignored files or a developer secret manager.

### Python quality gates

- `uv` or Poetry, with `uv` preferred for a new project
- Ruff formatter/linter
- mypy or Pyright
- pytest
- pytest-asyncio where needed
- coverage thresholds for important domain/security code

### TypeScript quality gates

- pnpm
- TypeScript strict mode
- ESLint
- Prettier
- Vitest/Jest
- Playwright for end-to-end UI tests

Crawler unit tests should use sanitized offline fixtures rather than real websites. Keep separate scheduled source-health smoke tests.

---

## 10. CI pipeline

Every PR should run roughly:

```text
checkout
  +-> secret detection
  +-> dependency review
  +-> Python format/lint/typecheck
  +-> TypeScript lint/typecheck
  +-> unit tests
  +-> integration tests
  +-> migration validation
  +-> crawler fixture tests
  +-> API contract tests
  +-> SAST / CodeQL
  +-> container build
  +-> container/IaC vulnerability scan
  +-> selected E2E tests
```

Recommended tooling:

- GitHub Actions
- CodeQL
- Dependency Review
- Dependabot or Renovate
- secret scanning where available
- Trivy or Grype
- optional Semgrep

Harden Actions with minimal permissions, immutable action pinning for sensitive workflows, careful handling of untrusted PR input and OIDC for cloud auth instead of long-lived cloud keys.

---

## 11. CI/CD and environments

Use three conceptual environments.

### Development

Developer machines, synthetic/test data, Compose dependencies and no production credentials.

### Staging

Production-like deployment, separate account/project/VPC where possible, sanitized or synthetic data, migration/deployment checks and E2E.

### Production

Tight IAM, real retention/access rules, restricted operator access, formal backup/restore and audit enabled.

Build an artifact once and promote the same artifact:

```text
PR checks
  -> merge main
  -> build image
  -> generate SBOM
  -> sign/attest image
  -> push registry
  -> deploy staging
  -> smoke/integration checks
  -> approval gate where required
  -> deploy production
```

Tag production releases and retain the Git commit SHA in image metadata.

---

## 12. Infrastructure as code

Use Terraform or OpenTofu from the beginning for shared environments.

Keep modules simple:

```text
network
postgres
object-storage
redis/queue
container-runtime
registry
identity integration
observability
kms/secrets
```

Require PR review for infrastructure changes and run formatting, validation, plan and IaC security scanning in CI.

### Kubernetes

Do **not** start with Kubernetes unless the organization already has a mature platform team and cluster.

Move there only when worker pools, autoscaling, many independent services, scheduling needs or existing operational standards justify it.

---

## 13. Database migrations

Use Alembic migrations committed with the code that requires them.

Rules:

- never edit a released migration
- prefer backward-compatible expand/migrate/contract changes
- back up before destructive migrations
- exercise migrations in CI against older schema versions
- keep rollout ordering in mind

Safe pattern:

```text
1. Add nullable/new field.
2. Deploy code that writes both forms.
3. Backfill.
4. Switch readers.
5. Remove old field in a later release.
```

---

## 14. Supply-chain security

Recommended controls:

- lock dependencies
- review dependency additions in PRs
- automated update PRs
- SBOM per release
- signed/attested build artifacts
- scan base images
- minimize OS packages
- pin production images by digest
- avoid arbitrary install scripts in CI
- restrict workflow-file modification
- use CODEOWNERS for workflows, infra, auth/security code and migrations

---

## 15. Observability

Instrument with OpenTelemetry.

Track collection metrics such as request rate, response codes, latency, bytes, retries, queue depth, browser fallback percentage and duplicate percentage.

Track pipeline metrics such as normalization latency, enrichment latency, AI failures, indexing lag and alert lag.

Track application metrics such as API latency/error rate, database saturation, search latency and auth failures.

Track security signals such as blocked internal URL fetches, blocked redirects, malware detections, anomalous export volume and permission failures.

Use structured correlation IDs such as:

```text
request_id
job_id
crawl_id
signal_id
case_id
```

Do not dump raw sensitive content into ordinary logs.

---

## 16. Backup and recovery

Back up:

- PostgreSQL
- object storage according to retention requirements
- infrastructure state
- critical configuration
- search indexes only where rebuilding is too expensive

Practice restore tests. Search indexes and embeddings should normally be reconstructible from PostgreSQL plus raw evidence.

---

## 17. Development phases

### Phase 0: foundations

- threat model
- source/legal policy model
- Signal schema
- provenance schema
- auth architecture
- PostgreSQL + object storage
- Docker development environment
- CI security gates

### Phase 1: useful MVP

- source configuration
- static HTTP/Scrapy collector
- raw capture + hashes
- normalization
- Signal list/detail UI
- basic search/filtering
- cases/tags/notes
- audit
- scheduled crawling

### Phase 2: richer collection

- isolated Playwright workers
- RSS/API connectors
- source-health dashboards
- WARC/screenshot support
- better retry/backoff/deduplication
- retention policies

### Phase 3: AI enrichment

- language detection
- translation
- entities
- embeddings
- similarity search
- summarization
- topic/relevance classifiers
- model/prompt provenance
- evaluation datasets

### Phase 4: fusion

- narrative clusters
- cross-source correlation
- entity/account relationship model
- graph UI
- watchlists/alerts
- case timelines

### Phase 5: production hardening and scale

- dedicated search cluster if justified
- durable workflows if justified
- dedicated egress infrastructure
- stronger ABAC/tenant isolation
- HA and DR
- worker autoscaling
- Kubernetes only when justified

---

## 18. Deliberately defer

Do not build these into the first version:

- a custom distributed crawler framework
- Kafka without a demonstrated need
- Kubernetes from day one
- a graph database before graph traversal is a real workload
- a database per microservice
- custom authentication
- a custom vector database
- model fine-tuning before an evaluation dataset exists
- autonomous final judgments on sensitive classifications
- live rendering of arbitrary hostile HTML inside the analyst app
- complex anti-bot evasion infrastructure

---

## 19. Alternative stacks

### Option A: recommended balanced stack

```text
Next.js/TypeScript
FastAPI/Python
Scrapy + Playwright
PostgreSQL + pgvector
S3/MinIO
Redis + Dramatiq/Celery
Docker Compose -> simple container platform
Terraform/OpenTofu
GitHub Actions
OpenTelemetry
```

Best when you want a strong blend of scraping, AI, web productivity and coding-assistant effectiveness.

### Option B: TypeScript-heavy

```text
Next.js/TypeScript
NestJS/TypeScript
Crawlee + Playwright
PostgreSQL
S3
BullMQ/Redis
```

Advantage: fewer languages and shared types. Disadvantage: weaker Python-style scraping/data/AI ecosystem.

### Option C: Go service core + Python analysis

```text
React/TypeScript
Go API/ingestion services
Python Scrapy/AI workers
PostgreSQL
NATS/RabbitMQ
S3
```

Good when network-service throughput is already known to be large.

### Option D: enterprise .NET

```text
React/TypeScript
ASP.NET Core/C#
Python collection/AI workers
PostgreSQL or SQL Server
RabbitMQ
```

Good when organizational identity, logging and deployment are already centered on Microsoft/.NET.

---

## 20. Decision matrix

| Choice | Scraping ecosystem | AI ecosystem | Web productivity | Static safety | Coding-assistant practicality | Operational simplicity | Overall fit |
|---|---:|---:|---:|---:|---:|---:|---:|
| Python + TypeScript | 5 | 5 | 5 | 4 | 5 | 4 | **5** |
| TypeScript only | 4 | 3 | 5 | 5 | 5 | 5 | **4** |
| Go + Python + TS | 5 | 5 | 5 | 5 | 4 | 3 | **4** |
| Rust + TS | 3 | 3 | 5 | 5 | 3 | 2 | **3** |
| C# + Python + TS | 5 | 5 | 5 | 5 | 5 | 3 | **4** |

Recommendation: **TypeScript + Python**, with Go or Rust added later only where there is a demonstrated reason.

---

## 21. Initial engineering rules

1. Raw evidence is immutable; analysis is versioned.
2. Every derived claim links back to captures.
3. Browser rendering is isolated from trusted infrastructure.
4. The API never fetches arbitrary external URLs itself.
5. Production data does not appear in developer fixtures by default.
6. No secrets in Git.
7. `main` is protected and deployable.
8. Every PR passes formatting, typing, tests and security checks.
9. Every database change is a migration.
10. Every important architecture/security decision gets an ADR.
11. Build once and promote the same artifact through environments.
12. AI classifications remain reviewable evidence-backed outputs.
13. Source collection policies are explicit and auditable.
14. Do not add infrastructure merely because it is fashionable.

---

## 22. References

### Crawling and browser automation

- Scrapy architecture: https://docs.scrapy.org/en/latest/topics/architecture.html
- Scrapy documentation: https://docs.scrapy.org/en/latest/
- Scrapy asyncio support: https://docs.scrapy.org/en/master/topics/asyncio.html
- Playwright Python: https://playwright.dev/python/docs/library
- Playwright browsers: https://playwright.dev/python/docs/browsers

### Data and search

- PostgreSQL Row-Level Security: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- OpenSearch vector search: https://docs.opensearch.org/latest/vector-search/

### Security

- OWASP SSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP XSS Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- GitHub Actions security: https://docs.github.com/en/actions/how-tos/secure-your-work
- GitHub Actions OIDC: https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers

### Observability

- OpenTelemetry: https://opentelemetry.io/docs/

### Language and AI-development context

- GitHub Octoverse 2025: https://octoverse.github.com/
- GitHub language/AI development overview: https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/

---

The project should be treated as a **secure signal-collection and fusion platform**, not merely a scraper with an AI dashboard.

```text
TypeScript analyst UI
        |
Python API + PostgreSQL
        |
Job queue
        |
Isolated Scrapy / Playwright workers
        |
Raw evidence store
        |
Normalization -> Signal
        |
AI/search enrichment
        |
Correlation / cases / alerts
```

This keeps the initial system straightforward while leaving clear upgrade paths for search scale, durable workflows, graph workloads and container orchestration.