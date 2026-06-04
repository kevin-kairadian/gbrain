# GBrain Operator Primer

First version. Verified against this repository on 2026-06-04.

This guide is for the person responsible for making GBrain useful in a real
operation. It assumes business fluency: ownership, permissions, workflows,
vendors, risk, and controls. It does not assume fluency with databases, command
line tools, OAuth, MCP, or software architecture.

The most important operating rule is simple: there is no meaningful standard
off-the-shelf GBrain for serious use. A useful brain needs deliberate choices
about business objects, page types, sources, access rules, schemas, skills,
ingestion, maintenance, cost, and hosting.

## Source Authority

Trust the source code over prose documentation when they differ. This primer
uses the required docs and the implementation files that control page parsing,
schema packs, sync, source routing, MCP, OAuth, storage, search, and operations.

Core repo sources:
[AGENTS.md](../AGENTS.md),
[CLAUDE.md](../CLAUDE.md),
[llms.txt](../llms.txt),
[brains and sources](architecture/brains-and-sources.md),
[system of record](architecture/system-of-record.md),
[schema packs](architecture/schema-packs.md),
[schema author tutorial](schema-author-tutorial.md),
[what schemas unlock](what-schemas-unlock.md),
[MCP deploy](mcp/DEPLOY.md),
[storage tiering](storage-tiering.md),
[engines](ENGINES.md),
[live sync](guides/live-sync.md),
[search modes](guides/search-modes.md),
[skill resolver](../skills/RESOLVER.md),
[brain routing](../skills/conventions/brain-routing.md), and
[schema evolution](../skills/conventions/schema-evolution.md).

Core implementation sources:
[markdown parsing](../src/core/markdown.ts),
[import pipeline](../src/core/import-file.ts),
[schema-pack manifest](../src/core/schema-pack/manifest-v1.ts),
[schema-pack loader](../src/core/schema-pack/loader.ts),
[active pack resolution](../src/core/schema-pack/load-active.ts),
[schema-pack registry](../src/core/schema-pack/registry.ts),
[source resolver](../src/core/source-resolver.ts),
[source operations](../src/core/sources-ops.ts),
[sync](../src/commands/sync.ts),
[operation contract](../src/core/operations.ts),
[HTTP MCP server](../src/commands/serve-http.ts),
[OAuth provider](../src/core/oauth-provider.ts),
[scope hierarchy](../src/core/scope.ts),
[config loader](../src/core/config.ts),
[storage interface](../src/core/storage.ts),
[S3 storage](../src/core/storage/s3.ts),
[Supabase storage](../src/core/storage/supabase.ts),
[local storage](../src/core/storage/local.ts),
[database schema](../src/schema.sql),
[facts fence](../src/core/facts-fence.ts),
[takes fence](../src/core/takes-fence.ts),
[link extraction](../src/core/link-extraction.ts),
[write-through](../src/core/write-through.ts),
[search mode](../src/core/search/mode.ts), and
[search visibility SQL](../src/core/search/sql-ranking.ts).

Official Google Cloud sources used for GCP claims:
[Cloud Run](https://cloud.google.com/run/docs/overview/what-is-cloud-run),
[Cloud Run jobs](https://cloud.google.com/run/docs/create-jobs),
[GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview),
[Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres),
[Cloud SQL PostgreSQL extensions](https://cloud.google.com/sql/docs/postgres/extensions),
[Cloud SQL from Cloud Run](https://cloud.google.com/sql/docs/postgres/connect-run),
[Cloud SQL backups](https://cloud.google.com/sql/docs/postgres/backup-recovery/backups),
[Cloud Storage](https://cloud.google.com/storage/docs/introduction),
[Cloud Storage interoperability](https://cloud.google.com/storage/docs/interoperability),
[Cloud Storage HMAC keys](https://cloud.google.com/storage/docs/authentication/hmackeys),
[Secret Manager](https://cloud.google.com/secret-manager/docs/overview),
[Cloud Run secrets](https://cloud.google.com/run/docs/configuring/services/secrets),
[Cloud Scheduler](https://cloud.google.com/scheduler/docs/overview),
[IAM service accounts](https://cloud.google.com/iam/docs/service-account-overview),
[Cloud Run service identity](https://cloud.google.com/run/docs/securing/service-identity),
[Cloud Logging](https://cloud.google.com/logging/docs/overview), and
[Cloud Monitoring](https://cloud.google.com/monitoring/docs/monitoring-overview).

---

## 1. What GBrain Is

### Plain-language model

| Term | Operator meaning | Verified behavior |
| --- | --- | --- |
| Brain | The active knowledge base. In code, this is a database target: local PGLite, Postgres, or a mounted remote brain. | A brain is selected separately from source routing. See [brains and sources](architecture/brains-and-sources.md) and `OperationContext.brainId` in [operations](../src/core/operations.ts). |
| Source | A named content boundary inside one brain, normally tied to one repository or imported content stream. | `sources.id` scopes pages, files, facts, OAuth clients, and search filters. See [schema](../src/schema.sql) and [source resolver](../src/core/source-resolver.ts). |
| Page | A Markdown document plus a derived database row. | `parseMarkdown` reads frontmatter and body; `importFromContent` stores the parsed page, chunks, tags, aliases, provenance, and versions. |
| Schema pack | The business vocabulary for pages and links. | Packs define `page_types`, `path_prefixes`, aliases, link rules, frontmatter links, extractability, expert routing, and filing rules. See [manifest-v1](../src/core/schema-pack/manifest-v1.ts). |
| Skill | An instruction file that tells agents how to perform a workflow. | Skills route behavior and reduce agent judgment. See [skills resolver](../skills/RESOLVER.md). |
| Sync | The process that reads source files and updates the database. | `gbrain sync` walks syncable files, imports them, uses locks and checkpoints, and updates `sources.last_commit`. See [sync](../src/commands/sync.ts). |
| Database | A searchable, indexed, operational copy of the brain. | Markdown is canonical for pages; DB also holds operational state such as tokens, jobs, request logs, raw data, and file metadata. See [system of record](architecture/system-of-record.md). |
| System of record | The place treated as authoritative. | For normal pages, it is the Markdown source repo. The DB is derived, except for DB-only operational data. |

### Operator decisions

Decide:

- What business or personal purpose each brain serves.
- Whether one brain is enough, or separate brains are needed for stronger boundaries.
- Which source repos exist inside each brain.
- Which schema pack is active for each brain or source.
- Which skills define operating workflows.
- Which data lives in Markdown and which data is DB-only.

### Native GBrain

GBrain natively provides brain/source routing, Markdown page parsing, schema-pack
resolution, sync, search, tags, links, facts, takes, timeline entries, raw data,
file metadata, MCP operations, OAuth clients, and local/remote caller
distinctions.

### Must be configured

Database engine, source repos, active schema pack, API keys, search mode,
embedding model, sync schedule, hosted MCP settings, OAuth clients, and storage
backend are operator choices.

### Requires extension or company instructions

Business-specific page templates, role policies, approval workflows, vendor
collectors, domain schemas, tenant lifecycle processes, and human authoring
rules are not universal GBrain behavior. They must be written as schema packs,
skills, docs, scripts, or external services.

---

## 2. Implementation Patterns

| Pattern | Use when | Boundary choice | Native GBrain | Must configure | Extend or document |
| --- | --- | --- | --- | --- | --- |
| Personal brain | One person owns the brain and most data. | One brain, one default source, optional topic sources. | PGLite default, local CLI, Markdown repo, search, facts, takes. | Search mode, model keys, schema pack, sync. | Personal authoring rules and ingestion habits. |
| Family brain | Multiple trusted family members share knowledge. | One brain, source per member or domain if needed. | Sources, OAuth clients, source-scoped reads/writes, visibility flags for facts/takes. | Per-member clients, shared schema, privacy rules. | Family policy for private facts, minors, shared devices, and deletion. |
| Internal company brain | One company uses GBrain internally. | One company brain; sources per department, system, or repo. | Multi-source routing, OAuth scopes, schema packs, skills, logs. | Source ownership, schema pack, skills, hosted MCP, backups. | Approval workflow, onboarding/offboarding, data retention, incident process. |
| Multi-source company brain | Departments need separable corpora but shared retrieval. | One brain with multiple sources and selected federation. | `sources.config.federated` and per-source routing. | Which sources federate by default, OAuth client source scopes. | Cross-source citation and escalation rules. |
| Multi-tenant SaaS | Each customer/company needs its own brain. | Strongest default: one brain/database/deployment per tenant. | Brains are database boundaries; sources are in-brain boundaries. | Tenant provisioning, secrets, backups, OAuth clients, environments. | Tenant router, billing, isolation audits, offboarding automation. |
| Industry-specific brain | A domain has stable objects and workflows. | One brain per organization; domain schema pack per industry. | Schema packs, link rules, extractability, expert routing. | Domain page types and path prefixes. | Domain skills, examples, collectors, controls, reports. |

Open documentation issue: The repo supports source-scoped OAuth and
multi-source search, but it does not document a complete shared-database
multi-tenant SaaS isolation model. Treat separate customer brains/databases as
the deterministic tenant boundary until a tenant isolation design is documented
and tested.

Sources:
[brains and sources](architecture/brains-and-sources.md),
[brain routing](../skills/conventions/brain-routing.md),
[sources schema](../src/schema.sql),
[OAuth provider](../src/core/oauth-provider.ts),
[operations source scope](../src/core/operations.ts).

---

## 3. Operating Boundaries

| Boundary | What it protects | What crosses it | Operator rule |
| --- | --- | --- | --- |
| Brain | Database, secrets, operational state, derived indexes. | Cross-brain federation is agent-decided, not deterministic database fan-out. | Use separate brains when the trust boundary is strong. |
| Source repo | A content corpus inside one brain. | Federated search can read across selected sources. | Use sources for departments, repos, systems, or customer subareas inside a trusted brain. |
| Tenant | A customer/company boundary. | Nothing should cross without contract and audit. | Prefer one brain per tenant for SaaS. Do not rely on source boundaries alone for hostile or regulated tenants. |
| Environment | Development, staging, production. | Schema changes and content may be promoted intentionally. | Use separate GCP projects, databases, secrets, source repos, and OAuth clients. |
| Local vs remote caller | Trusted machine owner vs untrusted agent-facing path. | Remote MCP calls use operation scopes and source context. | Run sensitive filesystem and maintenance commands locally or in trusted jobs. |

Native GBrain:

- `OperationContext.remote=false` is trusted local CLI.
- `OperationContext.remote=true` is untrusted agent-facing MCP.
- `sourceScopeOpts` threads source filtering for most read-side operations.
- Search visibility hides soft-deleted pages, archived sources, and quarantined pages.

Must configure:

- Source IDs and ownership.
- Which sources are federated.
- OAuth client source scopes.
- Separate environments and secrets.

Requires extension or documentation:

- Tenant-specific legal isolation.
- Environment promotion workflow.
- Exception handling for cross-source reads.

Open documentation issue: In [operations](../src/core/operations.ts), the
`query` operation accepts a `source_id` parameter and the special value
`__all__`. Source comments frame this as local cross-source search, but the
handler branch should be audited before treating source-scoped OAuth as a
complete tenant isolation boundary for remote query clients.

Sources:
[brains and sources](architecture/brains-and-sources.md),
[brain routing](../skills/conventions/brain-routing.md),
[operation context](../src/core/operations.ts),
[search visibility](../src/core/search/sql-ranking.ts).

---

## 4. Access, Authentication, And Authorization

### Concepts

| Concept | Plain meaning | GBrain behavior |
| --- | --- | --- |
| Human | A person using CLI, UI, or a connected client. | GBrain has OAuth clients and tokens, but this repo is not a full workforce identity product. |
| Agent | An AI client or automation calling tools. | MCP operations expose a tool surface; HTTP MCP requires OAuth. |
| Local CLI | Commands run by the machine owner or trusted job. | Trusted for filesystem access and protected operations. |
| Remote MCP | Tool calls over stdio or HTTP. | Treated as untrusted; HTTP path checks OAuth scopes. |
| OAuth | A token system for granting clients limited access. | GBrain supports client registration, authorization code with PKCE, client credentials, refresh, revocation, and legacy token fallback. |
| Scope | A permission label. | GBrain scopes are `read`, `write`, `admin`, `sources_admin`, `users_admin`, and `agent`. |
| Source scope | Which source a client writes to and which sources it can read. | OAuth clients store `source_id` and `federated_read`. |

### Operator decisions

Decide:

- Which humans can administer the brain.
- Which agents can read, write, manage sources, manage users, or submit agent jobs.
- Which source each client can write.
- Which sources each client can read.
- Whether legacy bearer tokens are allowed during migration.
- How private facts and takes are handled.

### Native GBrain

GBrain provides:

- Scope hierarchy: `admin` implies `read`, `write`, `sources_admin`, and `users_admin`; `write` implies `read`; `agent` is separate and not implied by `admin`.
- OAuth clients with token TTLs, source IDs, and federated read lists.
- Remote MCP scope checks before operation dispatch.
- Remote request logs in `mcp_request_log`.
- Redacted request parameter logging by default.
- Remote `get_page` stripping of private facts and takes fences.
- Remote fact trajectory reads filtered to `visibility='world'`.
- Local-only operations hidden from HTTP MCP tools.

### Must be configured

Configure:

- OAuth clients per agent, integration, or human-facing app.
- Scopes per client.
- Source write scope and federated read list.
- Token rotation and revocation process.
- Private vs world visibility policy for facts and takes.
- Admin bootstrap token handling for `gbrain serve --http`.

### Requires extension or company instructions

GBrain does not by itself define your company’s job roles, approval chain,
customer admin model, HR offboarding process, or vendor access policy. Write
those as operating procedures and map them to GBrain OAuth clients and scopes.

Open documentation issue: Human identity integration beyond GBrain OAuth clients
is not fully documented in the inspected repo. If you need SSO, SCIM, workforce
identity, or customer admin portals, design and test that layer explicitly.

Sources:
[MCP deploy](mcp/DEPLOY.md),
[OAuth provider](../src/core/oauth-provider.ts),
[scope hierarchy](../src/core/scope.ts),
[HTTP MCP server](../src/commands/serve-http.ts),
[operation contract](../src/core/operations.ts),
[facts fence](../src/core/facts-fence.ts),
[takes fence](../src/core/takes-fence.ts).

---

## 5. GCP Hosting Model

GBrain does not ship a confirmed first-party GCP deployment blueprint in the
inspected docs. The following model is an implementation option for production
on Google Cloud Platform, not built-in GBrain behavior.

### Recommended GCP shape

| Need | GCP option | Confirmed GBrain behavior | Operator decision |
| --- | --- | --- | --- |
| HTTP MCP service | [Cloud Run](https://cloud.google.com/run/docs/overview/what-is-cloud-run) service running `gbrain serve --http`. | GBrain exposes HTTP MCP, OAuth, health, admin routes. | Decide ingress, public URL, service identity, and whether Cloud Run’s disposable filesystem fits your source checkout model. |
| Scheduled maintenance | [Cloud Run jobs](https://cloud.google.com/run/docs/create-jobs), [Cloud Scheduler](https://cloud.google.com/scheduler/docs/overview), or GKE cron. | GBrain has CLI commands for sync, embed, extract, doctor, migrations, and jobs. | Decide whether schedules call HTTP endpoints, run container jobs, or run in GKE. Make repeated executions idempotent. |
| Long-running workers or persistent checkouts | [GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview) or another persistent host. | `gbrain sync` expects a local repo path or configured source path. | Use GKE or another persistent host when the source repo checkout must persist across runs. |
| Postgres database | [Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres) if extensions and connection model pass tests. | GBrain supports Postgres and uses `vector`, `pg_trgm`, and `pgcrypto` in schema bootstrap. | Validate Cloud SQL extension support, connection path, migrations, pooling, and backup restore before production. |
| Database connection from Cloud Run | [Cloud SQL from Cloud Run](https://cloud.google.com/sql/docs/postgres/connect-run). | GBrain accepts `GBRAIN_DATABASE_URL` or `DATABASE_URL`; env URL means Postgres. | Choose socket, connector, or private IP path and test migrations. |
| File/object storage | GBrain-native: local, S3-compatible, Supabase Storage. GCP option: Cloud Storage through a custom adapter or S3-compatible XML API path. | No native `gcs` storage backend exists in [storage.ts](../src/core/storage.ts). | Do not claim Cloud Storage is built in. Test any Cloud Storage interoperability path before using it for production files. |
| Secrets | [Secret Manager](https://cloud.google.com/secret-manager/docs/overview) and [Cloud Run secrets](https://cloud.google.com/run/docs/configuring/services/secrets). | GBrain reads keys and DB URLs from config and environment variables. | Store DB URLs, API keys, OAuth secrets, bootstrap tokens, and HMAC keys as secrets. |
| Service identity | [IAM service accounts](https://cloud.google.com/iam/docs/service-account-overview) and [Cloud Run service identity](https://cloud.google.com/run/docs/securing/service-identity). | GBrain does not assign GCP roles. | Create least-privilege service accounts per environment and tenant. |
| Logging | [Cloud Logging](https://cloud.google.com/logging/docs/overview). | GBrain logs MCP request records in DB and writes process logs. | Route service logs to Cloud Logging and decide retention. |
| Monitoring | [Cloud Monitoring](https://cloud.google.com/monitoring/docs/monitoring-overview). | GBrain has health/status/doctor surfaces. | Create uptime, error, latency, job, sync, and spend alerts. |
| Backups | [Cloud SQL backups](https://cloud.google.com/sql/docs/postgres/backup-recovery/backups), Git backups, object-store backups. | Markdown pages can be rebuilt; DB-only state cannot be rebuilt from Markdown alone. | Back up source repos, schema packs, config, DB-only tables, files, and secrets. |

### GCP-specific cautions

- Cloud Run instances have a disposable writable filesystem. Do not rely on a
  Cloud Run service instance as the only durable checkout of a source repo.
- Cloud Run jobs run to completion and exit. They fit migrations, sync jobs,
  extraction jobs, and doctor checks better than always-on maintenance loops.
- Cloud Scheduler has at-least-once delivery. Scheduled GBrain endpoints or
  jobs must be safe to retry.
- Cloud SQL is a Postgres option, not a documented GBrain-specific database
  target. Prove extensions, migrations, vector indexes, search, backups, and
  restore before launch.
- Cloud Storage buckets store objects, but GBrain’s storage interface does not
  include a native Cloud Storage backend. Cloud Storage’s XML API can interop
  with some S3 tools using HMAC keys; this requires testing with GBrain’s
  S3-compatible backend before production.

Open documentation issue: No inspected repo document provides a complete GCP
reference architecture, Terraform module, Cloud Run service definition, Cloud
Run job definition, Cloud SQL compatibility matrix, or Cloud Storage adapter
test report.

Sources:
[MCP deploy](mcp/DEPLOY.md),
[engines](ENGINES.md),
[config](../src/core/config.ts),
[schema](../src/schema.sql),
[storage](../src/core/storage.ts),
[Cloud Run](https://cloud.google.com/run/docs/overview/what-is-cloud-run),
[Cloud SQL extensions](https://cloud.google.com/sql/docs/postgres/extensions),
[Cloud Storage interoperability](https://cloud.google.com/storage/docs/interoperability).

---

## 6. Storage And System Of Record

### Operator decisions

Decide:

- Which Git repo is the canonical Markdown source for each source.
- Which data is written as Markdown pages.
- Which data is stored as raw DB evidence.
- Which files stay in Git and which go to object storage.
- Which DB-only data must be backed up.
- How provenance is stamped for each ingestion path.

### Native GBrain

GBrain uses Markdown plus YAML frontmatter as the normal page source of
record. The database stores derived page rows, chunks, tags, links, timeline
entries, versions, and search indexes. Facts and takes are represented by
Markdown fences and reconciled into DB tables. Raw data, OAuth data, jobs,
request logs, config rows, file metadata, and operational state are DB-only.

The import path:

1. Parses Markdown/frontmatter.
2. Applies size and content sanity guards.
3. Computes a content hash.
4. Stores a page version.
5. Upserts the page.
6. Adds tags.
7. Chunks and embeds when configured.
8. Records aliases and provenance.

`put_page` can write through to the source repo when `sync.repo_path` is
configured. The write-through renders from the DB row and atomically renames a
temporary file into place.

### Must be configured

Configure:

- `sync.repo_path` or source-specific local paths.
- Source IDs and source repos.
- Storage backend if files are uploaded out of Git.
- Backup policy for Markdown repos and DB-only data.
- Provenance conventions for collectors and agents.

### Requires extension or company instructions

Collectors for CRM, email, storage drives, meetings, support tickets, or
industry feeds are business-specific. They should write pages, raw data, and
provenance deterministically.

Open documentation issue: `gbrain files signed-url` uses the configured storage
backend, but the MCP `file_url` operation currently returns a `gbrain:files/...`
placeholder with a TODO for signed URLs. Operators should test file retrieval
for their chosen backend before promising end-user file URLs.

Sources:
[system of record](architecture/system-of-record.md),
[storage tiering](storage-tiering.md),
[import pipeline](../src/core/import-file.ts),
[write-through](../src/core/write-through.ts),
[files command](../src/commands/files.ts),
[file operations](../src/core/operations.ts),
[storage interface](../src/core/storage.ts).

---

## 7. Configuration Model

### Operator decisions

| Choice | Options confirmed in repo | Decision rule |
| --- | --- | --- |
| Database | PGLite or Postgres. | Use PGLite for personal/local use. Use Postgres for hosted, multi-user, large, or synchronized operation. |
| Postgres host | Repo documents Supabase and self-hosted Postgres; GCP Cloud SQL is an implementation option. | Use Cloud SQL only after proving migrations, extensions, and performance. |
| Storage backend | `local`, `s3`, `supabase`. | Use local for testing, S3-compatible or Supabase for production files. Treat Cloud Storage as custom/interop until tested. |
| Model keys | OpenAI, Anthropic, ZeroEntropy and other configured providers via model gateway. | Store keys in Secret Manager or another secret store, not in source repos. |
| Embeddings | Configured by `embedding_model`, dimensions, columns, and provider keys. | Fix dimensions before production indexing; changing embeddings can require reindexing. |
| Search mode | `conservative`, `balanced`, `tokenmax`. | Confirm with the operator after init because cost spread is material. |
| Schema pack | Active pack from per-call local option, env, source config, DB config, `gbrain.yml`, file config, or default. | Set the pack at brain or source level and document override precedence. |
| Scheduling | Host cron, Cloud Scheduler, Cloud Run jobs, GKE cron, or GBrain jobs. | Use idempotent scheduled jobs with logs and retry rules. |
| Hosted operation | `gbrain serve --http` for remote MCP. | Require OAuth, public URL, TLS/reverse proxy, scopes, and request logging. |
| Secrets | Environment variables and config file are read by GBrain. | Inject secrets through GCP Secret Manager or equivalent. |
| Environments | Not a single GBrain switch. | Use separate projects, DBs, repos, secrets, and OAuth clients for dev/staging/prod. |

### Native GBrain

`loadConfig` reads `~/.gbrain/config.json` and environment overrides such as
`GBRAIN_DATABASE_URL`, `DATABASE_URL`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
`ZEROENTROPY_API_KEY`, and model/search-related env vars. A database URL
forces Postgres mode.

### Must be configured

Every production brain needs explicit settings for database, source repos,
schema pack, search mode, model keys, embedding provider, storage backend,
OAuth clients, maintenance schedule, backups, and environment names.

### Requires extension or company instructions

Cost approvals, model-vendor risk review, environment promotion, key rotation,
and incident response are operator processes, not built-in schema behavior.

Sources:
[AGENTS search-mode stop](../AGENTS.md),
[INSTALL_FOR_AGENTS](../INSTALL_FOR_AGENTS.md),
[engines](ENGINES.md),
[search modes](guides/search-modes.md),
[config](../src/core/config.ts),
[search mode implementation](../src/core/search/mode.ts),
[storage](../src/core/storage.ts).

---

## 8. Schema Pack Primer

Schema packs define the vocabulary and routing rules for the brain. They do
not define complete page templates.

### Operator decisions

Decide:

- Which business objects deserve first-class page types.
- Which folders map to each page type.
- Which page types are aliases for search and expert routing.
- Which frontmatter fields create links.
- Which link verbs matter.
- Which page types are extractable into facts.
- Which page types should surface as experts.
- Which filing rules agents should follow.
- Which pack is a starter pack and which is a domain pack.

### Native GBrain

Schema-pack manifests support:

- `api_version`
- `name`
- `version`
- `gbrain_min_version`
- `extends`
- `borrow_from`
- `page_types`
- `link_types`
- `frontmatter_links`
- `takes_kinds`
- `enrichable_types`
- `filing_rules`
- `phases`
- `calibration_domains`
- `migration_from`
- `mapping_rules`

Each `page_types[]` entry can include:

- `name`
- `primitive`
- `path_prefixes`
- `aliases`
- `extractable`
- `expert_routing`
- `subtypes`

The base pack includes starter page types such as `person`, `company`,
`meeting`, `note`, `source`, `project`, and `writing`. The recommended pack
extends the base pack with additional operational brain types.

### Must be configured

Set the active pack and keep pack files versioned. For custom packs, place
operator-owned packs outside bundled pack files and activate them through
config. Do not mutate bundled packs as the source of business truth.

### Requires extension or company instructions

Page templates, required sections, review rules, examples, and collection
workflows belong in operator docs and skills. Schema packs tell GBrain what a
type means for routing and extraction; they do not create a full authoring
contract by themselves.

Important rule: page frontmatter `type` should match the schema pack
`page_types[].name`. Folder or subfolder mapping is controlled by
`page_types[].path_prefixes`.

Sources:
[schema packs](architecture/schema-packs.md),
[schema author tutorial](schema-author-tutorial.md),
[schema evolution](../skills/conventions/schema-evolution.md),
[manifest-v1](../src/core/schema-pack/manifest-v1.ts),
[base pack](../src/core/schema-pack/base/gbrain-base.yaml),
[recommended pack](../src/core/schema-pack/base/gbrain-recommended.yaml),
[markdown type inference](../src/core/markdown.ts).

---

## 9. Designing A Domain Schema

Use this process for a company or industry pack.

### Deterministic process

1. List the business decisions the brain must support.
2. List the business objects: people, companies, assets, deals, accounts,
   vendors, mines, restaurants, permits, invoices, trades, funds, projects,
   meetings, incidents, or policies.
3. For each object, decide whether it deserves a page type:
   - Under 20 pages: use an existing type or a folder convention.
   - 20 to 100 pages: consider an alias or narrow prefix.
   - Over 100 pages or with distinct workflows: create a first-class type.
4. Choose `page_types[].name` values using stable business language.
5. Choose `path_prefixes` that map to real source folders.
6. Choose stable frontmatter fields.
7. Define link verbs that answer business questions.
8. Define `frontmatter_links` for fields that should become graph edges.
9. Mark `extractable` only for types where evidence-backed facts should be mined.
10. Mark `expert_routing` only for types that should answer “who knows?”
11. Add `filing_rules` for where agents should put new pages.
12. Write examples and a skill for creating and updating pages.
13. Run `schema sync --apply` only after reviewing impact.
14. Verify with import, search, `whoknows`, extraction, and sample questions.

### Native GBrain

GBrain can load, validate, resolve, lint, mutate, and activate schema packs.
It can use path prefixes for type inference, aliases for type closure, link
rules for graph construction, `extractable` for fact extraction surfaces, and
`expert_routing` for expert queries.

### Must be configured

Domain packs must be owned, versioned, reviewed, activated, and tested. Decide
which source uses which pack when a brain has multiple business domains.

### Requires extension or company instructions

Industry logic is not built in. A fintech brain, mining brain, restaurant
brain, services brain, or investing brain needs domain-specific page types,
examples, skills, validation rules, reports, and ingestion collectors.

Sources:
[schema evolution](../skills/conventions/schema-evolution.md),
[schema author tutorial](schema-author-tutorial.md),
[what schemas unlock](what-schemas-unlock.md),
[schema pack operations](../src/core/operations.ts),
[schema pack manifest](../src/core/schema-pack/manifest-v1.ts).

---

## 10. Page Model And Authoring Contract

### Operator decisions

Decide:

- Required frontmatter fields per page type.
- Required body sections per page type.
- Whether facts and takes are allowed on each type.
- What belongs in timeline vs body.
- What fields create links.
- How custom sections are named.
- How evidence and provenance are cited.
- Who can create, edit, approve, or delete pages.

### Native GBrain

Markdown pages use YAML frontmatter and a Markdown body. `parseMarkdown` treats
`type`, `title`, `tags`, and `slug` as first-class fields. It removes those
from the stored JSON frontmatter, while preserving other custom frontmatter.

Native page structures include:

- `type`
- `title`
- `tags`
- custom frontmatter
- body Markdown
- links from Markdown, wikilinks, and frontmatter link fields
- `## Facts` fence
- `## Takes` fence
- timeline content and structured timeline entries

### Must be configured

The operator must document page templates and examples. The schema pack only
defines type names, prefixes, aliases, link rules, extractability, expert
routing, and filing rules.

### Requires extension or company instructions

Required fields, page templates, approval rules, business-specific sections,
and examples must be implemented in docs, skills, lints, or custom collectors.

Important rule: frontmatter `type` should match a schema pack
`page_types[].name`. Folder or subfolder mapping is controlled by
`page_types[].path_prefixes`. Schema packs do not define complete page
templates.

Sources:
[markdown parser](../src/core/markdown.ts),
[facts fence](../src/core/facts-fence.ts),
[takes fence](../src/core/takes-fence.ts),
[link extraction](../src/core/link-extraction.ts),
[schema pack manifest](../src/core/schema-pack/manifest-v1.ts).

---

## 11. Page Creation Workflow

Use this workflow when turning a business object into a page.

1. Identify the source: which repo or content stream owns this object.
2. Identify the brain: which database boundary owns it.
3. Identify the page type: choose a `page_types[].name`.
4. Choose the path: use the matching `path_prefixes` rule.
5. Create a stable slug.
6. Write frontmatter:
   - `type`
   - `title`
   - `tags`
   - stable custom fields
   - link fields declared in `frontmatter_links`
7. Write the body using the operator’s page template.
8. Add Markdown links or wikilinks for explicit relationships.
9. Add facts only when the claim is evidence-backed and belongs on that entity.
10. Add takes only when the assertion needs holder, weight, and review history.
11. Add timeline entries for dated events.
12. Attach raw evidence or file references when needed.
13. Run import or sync.
14. Verify the page with direct get.
15. Verify retrieval with search/query.
16. Verify links, tags, facts, takes, and timeline if used.
17. Commit the Markdown source repo if this is a source-controlled page.

Native GBrain:

- Parses and imports the page.
- Computes content hash.
- Adds tags.
- Stores aliases.
- Chunks and embeds content when configured.
- Records provenance.
- Can write through from DB to Markdown for `put_page` if a source repo is configured.

Must be configured:

- Page type, source repo, active schema pack, authoring template, and review rules.

Requires extension or company instructions:

- Business object mapping, required sections, evidence policy, approval chain,
  and post-creation checks.

Sources:
[markdown parser](../src/core/markdown.ts),
[import pipeline](../src/core/import-file.ts),
[write-through](../src/core/write-through.ts),
[schema packs](architecture/schema-packs.md).

---

## 12. Native Knowledge Structures

| Structure | Native behavior | Operator use |
| --- | --- | --- |
| Tags | Stored per page and indexed. | Use for status, domain, sensitivity, workflow, or review labels. |
| Links | Stored graph edges with source/provenance fields. | Use for relationships such as works-at, founded, attended, invested-in, related-to. |
| Facts | Evidence-backed rows in a `## Facts` fence, reconciled to DB. | Use for claims about an entity, including typed metrics and events. |
| Takes | Beliefs or assertions with holder, kind, weight, dates, and resolution fields. | Use for opinions, bets, hypotheses, and evolving judgment. |
| Timeline | Dated events and structured timeline entries. | Use for chronological history. |
| Raw evidence | JSON stored in `raw_data`. | Use for API responses and collector evidence that should not be rewritten as prose. |
| Provenance | Source kind, source URI, ingested via, ingested at. | Use for auditability and duplicate analysis. |
| Visibility | Facts and takes support private/world-style visibility behavior. | Use to control what remote agents can see. |

### Native GBrain

The database schema includes `pages`, `sources`, `links`, `tags`, `raw_data`,
`timeline_entries`, `files`, OAuth tables, request logs, jobs, and more. Facts
and takes are maintained through migrations and fence parsers.

### Must be configured

Operators must define tag taxonomy, link vocabulary, fact visibility policy,
take holder policy, evidence source list, and timeline event conventions.

### Requires extension or company instructions

Business-specific scoring, compliance tags, domain metrics, retention classes,
or audit reports must be added through schema, skills, collectors, reports, or
external systems.

Sources:
[schema](../src/schema.sql),
[facts fence](../src/core/facts-fence.ts),
[takes fence](../src/core/takes-fence.ts),
[link extraction](../src/core/link-extraction.ts),
[operations](../src/core/operations.ts).

---

## 13. Skills And Agent Behavior

### Operator decisions

Decide:

- Which workflows agents may perform.
- Which skills are authoritative for each workflow.
- Which tasks require a human approval step.
- Which fields and pages agents may write.
- Which skills are allowed in production.

### Native GBrain

Skills are instruction files that route agent behavior. The resolver maps task
types to skill files. GBrain can publish skills over MCP when configured.
Skills reduce LLM judgment by giving agents explicit rules for ingestion,
maintenance, scheduling, reporting, schema changes, and domain work.

### What belongs in schema vs skill

| Put in schema pack | Put in skill or operator doc |
| --- | --- |
| Page type names. | Step-by-step workflow. |
| Folder prefixes. | Page templates and examples. |
| Link verbs. | Approval rules. |
| Frontmatter link fields. | Vendor-specific ingestion steps. |
| Extractability. | Review checklists. |
| Expert-routing flags. | Escalation and incident rules. |
| Filing rules. | Human communication. |

### Must be configured

Set which skills agents should use and how those skills are distributed,
reviewed, and updated. If skills are exposed over MCP, configure publishing
and access.

### Requires extension or company instructions

Every serious deployment needs custom skills for company-specific authoring,
ingestion, maintenance, schema evolution, support, reporting, and governance.

Sources:
[skill resolver](../skills/RESOLVER.md),
[schema packs](architecture/schema-packs.md),
[list/get skill operations](../src/core/operations.ts).

---

## 14. Ingestion Operations

### Operator decisions

Decide:

- Which sources are allowed to write pages.
- Which collectors are deterministic and which use LLM extraction.
- How source attribution is stamped.
- How duplicate content is detected.
- Which raw evidence is retained.
- Which files go to object storage.
- Which ingestion paths are remote-safe.

### Native GBrain

Confirmed native ingestion surfaces include:

- Markdown import and sync.
- `put_page` operation.
- `put_raw_data` and `get_raw_data`.
- File metadata and local-only file upload operations.
- Ingest logs.
- Provenance fields.
- Facts/takes extraction and reconciliation paths.
- Content hash duplicate checks.
- Remote trust gates for provenance and auto-linking.

Remote `put_page` cannot choose trusted provenance fields. The server stamps
remote writes as `mcp:put_page`. Local trusted callers can provide provenance.

### Must be configured

Configure source IDs, source repos, sync schedules, storage backend, API keys,
collector credentials, source attribution taxonomy, and duplicate policy.

### Requires extension or company instructions

Meetings, media, research, email, CRM, finance systems, mining systems,
restaurant systems, service-ticket systems, and investing feeds need custom
collectors or skills. A deterministic collector should write repeatable pages,
raw data, provenance, and files without relying on ad hoc agent judgment.

Open documentation issue: The repo has recipes and skills for ingestion
patterns, but there is no single product-level ingestion contract covering all
business systems. Each production source should get a source-specific runbook
and acceptance test.

Sources:
[import pipeline](../src/core/import-file.ts),
[operations](../src/core/operations.ts),
[files command](../src/commands/files.ts),
[storage interface](../src/core/storage.ts),
[system of record](architecture/system-of-record.md).

---

## 15. Search, Query, And Retrieval

### Operator decisions

Decide:

- Which search mode is acceptable for cost and quality.
- Which model tier downstream agents will use.
- Which sources are searched by default.
- Whether cross-source retrieval is allowed.
- Whether remote clients may use hybrid search.
- How retrieved answers are verified.

### Native GBrain

GBrain provides:

- Direct page retrieval.
- Keyword search.
- Hybrid search with chunks, embeddings, ranking, cache, title boost, graph signals, and optional reranking depending on mode/config.
- Three search modes: `conservative`, `balanced`, `tokenmax`.
- Per-call mode overrides for trusted local callers only.
- Remote search mode governed by server configuration.
- Source-scoped search for many operations.
- Search visibility filtering for soft-deleted pages, archived sources, and quarantined pages.

### Must be configured

Configure:

- `search.mode`.
- Embedding provider and dimensions.
- Search overrides only when justified.
- Source federation.
- Remote keyword-only mode if query text must not go to embedding providers.
- Evaluation captures if benchmarking retrieval changes.

### Verification rule

Search returns matching chunks or result summaries. For a business decision,
an operator or agent should retrieve the full page and inspect source evidence
before treating an answer as verified.

### Requires extension or company instructions

Domain-specific retrieval acceptance tests, benchmark queries, source
prioritization, citation rules, and answer-review policies are operator work.

Sources:
[search modes](guides/search-modes.md),
[CLAUDE search matrix](../CLAUDE.md),
[search mode implementation](../src/core/search/mode.ts),
[search/query operations](../src/core/operations.ts),
[brain routing](../skills/conventions/brain-routing.md).

---

## 16. Maintenance And Health

### Operator decisions

Decide:

- How often each source syncs.
- Which jobs run after sync: embed, extract, facts, links, timeline, doctor.
- Which health score or checks are required for launch.
- How migrations and upgrades are approved.
- How backups are restored and tested.
- Who responds to failed jobs.

### Native GBrain

GBrain provides:

- `gbrain sync` with locks, checkpoints, full/incremental paths, and source status.
- `gbrain doctor` checks and remediation options.
- `gbrain embed --stale` and extraction commands.
- Live sync guidance using cron or host schedulers.
- Migrations and `gbrain apply-migrations --yes`.
- Upgrade flow.
- Jobs and maintenance phases.
- RLS checks for Postgres/Supabase-style exposure.

### Must be configured

Configure:

- Sync cadence per source.
- Embedding and extraction cadence.
- Cron or Cloud Scheduler job definitions.
- Logs and alerts.
- Backup schedule.
- Restore drills.
- Migration window and rollback policy.

### Requires extension or company instructions

Define operational runbooks for failed sync, failed migrations, rising costs,
stale embeddings, broken links, missing facts, bad schema changes, and privacy
incidents.

GCP implementation:

- Use Cloud Scheduler only for idempotent jobs or endpoints because delivery is
  at least once.
- Use Cloud Run jobs for bounded maintenance tasks.
- Use Cloud SQL backups if Cloud SQL is the chosen Postgres host.
- Use Cloud Logging and Cloud Monitoring for service and job visibility.

Sources:
[live sync](guides/live-sync.md),
[before shipping](../AGENTS.md),
[sync implementation](../src/commands/sync.ts),
[doctor references](../src/commands/doctor.ts),
[migrations](../src/core/migrate.ts),
[Cloud Scheduler](https://cloud.google.com/scheduler/docs/overview),
[Cloud SQL backups](https://cloud.google.com/sql/docs/postgres/backup-recovery/backups).

---

## 17. Governance

### Operator decisions

Decide:

- Who can change schema packs.
- Who can add page types.
- Who can change access scopes.
- Who approves source federation.
- How tenants are onboarded and offboarded.
- What data is private, regulated, or retained.
- How audit logs are reviewed.
- What incident process applies to data leakage or bad writes.

### Native GBrain

GBrain provides schema-pack validation/mutation operations, source management,
OAuth scopes, source-scoped clients, request logs, soft delete, archive
behavior, facts/takes visibility, RLS posture for Postgres tables, and doctor
checks.

### Must be configured

Configure:

- Schema change review.
- Source owner list.
- OAuth client inventory.
- Permission recertification cadence.
- Retention policy.
- Backup retention.
- Tenant lifecycle checklist.
- Privacy rules.

### Requires extension or company instructions

Legal holds, data subject deletion, customer contract controls, SOC 2 evidence,
industry-specific retention, and incident communications must be implemented
outside the base schema.

Open documentation issue: Privacy policy is documented for public artifacts in
[CLAUDE.md](../CLAUDE.md), but a full operator data-governance policy for
company or SaaS deployments is not present in the inspected repo.

Sources:
[CLAUDE privacy](../CLAUDE.md),
[schema evolution](../skills/conventions/schema-evolution.md),
[RLS guide](guides/rls-and-you.md),
[OAuth provider](../src/core/oauth-provider.ts),
[source operations](../src/core/sources-ops.ts),
[operations](../src/core/operations.ts).

---

## 18. Deployment Playbooks

### Personal brain

1. Install and initialize.
2. Stop and confirm search mode with the operator.
3. Use PGLite unless hosted access is required.
4. Choose `gbrain-recommended` or a small custom pack.
5. Configure model keys and embeddings.
6. Choose one Markdown repo as the default source.
7. Set live sync or manual sync.
8. Run doctor and a search verification set.
9. Back up the repo and `~/.gbrain` state.

### Family brain

1. Decide shared vs private content rules.
2. Use one brain unless there are strong privacy boundaries.
3. Create sources per member or domain if helpful.
4. Create OAuth clients for trusted apps or agents.
5. Define visibility rules for facts and takes.
6. Document deletion and correction workflow.
7. Schedule sync and health checks.
8. Back up repo, database, and secrets.

### Company brain

1. Define business objectives and source owners.
2. Use Postgres for hosted operation.
3. Create separate dev/staging/prod environments.
4. Create sources per department, repo, or business system.
5. Build a company schema pack.
6. Write skills for ingestion, authoring, maintenance, and reporting.
7. Register OAuth clients with least privilege.
8. Deploy `gbrain serve --http`.
9. Schedule sync, embed, extract, doctor, and backup checks.
10. Run launch verification and permission review.

### Multi-tenant SaaS

1. Use one brain/database/deployment per customer unless a formal shared-DB isolation design is built and audited.
2. Create per-tenant GCP project or strongly separated resource set if required by contract.
3. Create per-tenant source repos, database, secrets, OAuth clients, service account, logs, and backups.
4. Automate tenant creation and offboarding.
5. Run schema and source checks during onboarding.
6. Run customer-specific launch verification.
7. Monitor sync, search, error, cost, and backup status per tenant.
8. Test delete/export/offboard before first production customer.

Sources:
[install protocol](../AGENTS.md),
[MCP deploy](mcp/DEPLOY.md),
[engines](ENGINES.md),
[schema author tutorial](schema-author-tutorial.md),
[Cloud Run](https://cloud.google.com/run/docs/overview/what-is-cloud-run),
[Cloud SQL](https://cloud.google.com/sql/docs/postgres).

---

## 19. Verification And Acceptance

### Launch checklist

- Brain boundary chosen.
- Source boundaries chosen.
- Schema pack active and versioned.
- Page type examples written.
- Skills written for core workflows.
- Search mode confirmed.
- Database engine selected.
- Embedding provider configured.
- Storage backend selected and tested.
- OAuth clients registered with least privilege.
- Remote MCP operations tested.
- Sync runs successfully.
- Embeddings are current.
- Links/facts/takes/timeline checks pass.
- Doctor passes or documented exceptions are accepted.
- Backup and restore tested.
- Logs and alerts configured.
- Tenant/environment separation verified.
- Cost guardrails reviewed.

### Ongoing checklist

- Review schema changes before applying.
- Review OAuth clients and scopes.
- Review source federation.
- Review failed sync/import files.
- Review stale embeddings and extraction lag.
- Review doctor output.
- Review backup success and restore drills.
- Review request logs for unusual agent activity.
- Review search quality with representative queries.
- Review cloud cost and model spend.

### Acceptance tests

For each production brain, prove:

1. A known page can be created from a business object.
2. The page lands in the expected source and path.
3. The page has the expected type.
4. Frontmatter links become expected graph links.
5. Facts/takes/timeline parse as expected.
6. Search finds the page from a realistic query.
7. Direct get returns the full page.
8. Remote read only sees allowed data.
9. Remote write only writes to allowed source/path.
10. Sync is idempotent.
11. Restore can recover the brain to a usable state.

Sources:
[doctor](../src/commands/doctor.ts),
[sync](../src/commands/sync.ts),
[operations](../src/core/operations.ts),
[search modes](guides/search-modes.md),
[testing](TESTING.md).

---

## 20. Appendices

### Appendix A: Glossary

| Term | Meaning |
| --- | --- |
| Agent | An AI or automation calling GBrain tools. |
| Brain | A database-backed knowledge boundary. |
| CLI | Command line interface; trusted when run locally by the operator. |
| Database | The indexed working store GBrain searches and updates. |
| Embedding | A numeric representation of text or media used for semantic search. |
| Frontmatter | YAML metadata at the top of a Markdown page. |
| Git repo | The versioned folder that holds canonical Markdown pages when a source is filesystem-backed. |
| MCP | Model Context Protocol; a tool interface used by agents. |
| OAuth | Token-based authorization for clients. |
| Page | A Markdown knowledge object plus its derived DB row. |
| PGLite | Local embedded Postgres-like engine used by default. |
| Postgres | Production relational database engine supported by GBrain. |
| Provenance | Where data came from and how it was ingested. |
| Schema pack | Business vocabulary and routing rules for page types and links. |
| Skill | Workflow instructions for agents. |
| Source | A named corpus/repo inside a brain. |
| System of record | The authoritative copy of data. |
| Tenant | A customer/company boundary in SaaS operation. |

### Appendix B: Configuration Checklist

- Brain name and purpose.
- Database engine.
- Database URL or PGLite path.
- Source IDs and repo paths.
- Active schema pack.
- Model keys.
- Embedding model and dimensions.
- Search mode.
- Storage backend.
- Hosted MCP public URL.
- OAuth clients and scopes.
- Sync schedule.
- Extraction schedule.
- Backup schedule.
- Log retention.
- Environment labels.
- Secret inventory.

### Appendix C: Schema Checklist

- Page types named.
- Primitives chosen.
- Path prefixes defined.
- Aliases reviewed.
- Link types defined.
- Frontmatter links mapped.
- Extractable types selected.
- Expert-routing types selected.
- Filing rules written.
- Examples written.
- Skills updated.
- Migration/backfill plan written.
- Search and extraction tests run.

### Appendix D: Page Authoring Checklist

- Correct source selected.
- Correct `type`.
- Correct path prefix.
- Stable title.
- Useful tags.
- Required frontmatter present.
- Link fields use resolvable slugs.
- Body follows operator template.
- Facts are evidence-backed.
- Takes have holder and weight.
- Timeline events have dates.
- Raw evidence attached or cited.
- Search verification done.
- Source repo committed if applicable.

### Appendix E: Access Matrix

| Actor | Recommended scope | Source boundary | Notes |
| --- | --- | --- | --- |
| Local operator | Local CLI or `admin` | All needed sources | Use for maintenance and filesystem operations. |
| Read-only agent | `read` | One source or federated read list | Cannot write pages. |
| Writing agent | `write` | One write source | `write` implies `read`; still restrict source. |
| Source admin | `sources_admin` | Brain-level source management | Does not imply user admin. |
| User admin | `users_admin` | OAuth/user management | Does not imply source admin. |
| Full admin | `admin` | All configured sources | Does not imply `agent`. |
| Agent dispatcher | `agent` | Bound tools/source/prefixes | Separate from admin by design. |
| SaaS customer agent | Prefer separate brain | Tenant brain only | Do not rely on shared source boundary without audit. |

### Appendix F: Tenant Onboarding Checklist

- Customer contract and data boundary recorded.
- Brain/database created.
- Source repos created.
- Schema pack selected.
- Tenant-specific config applied.
- Secrets created.
- Service account created.
- OAuth clients created.
- Backup configured.
- Logs and alerts configured.
- Initial pages imported.
- Search verified.
- Access verified.
- Offboarding path tested.

### Appendix G: Open Documentation Issues

- Open documentation issue: No complete GCP deployment reference architecture
  was found in the inspected repo. Check `docs/mcp/DEPLOY.md`, deployment
  scripts, and future infrastructure docs before presenting GCP support as
  built in.
- Open documentation issue: No native Cloud Storage backend exists in
  `StorageConfig`. Check `src/core/storage.ts` and storage tests before using
  Cloud Storage as a production file backend.
- Open documentation issue: Cloud Storage XML API/HMAC may interoperate with
  S3-style tools, but this repo does not document a tested GBrain S3 backend
  configuration for Cloud Storage. Test before production.
- Open documentation issue: Cloud SQL is not directly documented as a GBrain
  target. Check `src/schema.sql`, migrations, vector extension support, and
  connection behavior before production.
- Open documentation issue: Shared-database multi-tenant SaaS isolation is not
  documented. Use separate brains/databases/deployments until an isolation
  design is written and audited.
- Open documentation issue: `query.source_id='__all__'` needs explicit remote
  authorization documentation or enforcement review before source scope can be
  called a tenant isolation control.
- Open documentation issue: Human identity integration beyond OAuth clients is
  not specified. Design SSO/workforce/customer identity outside this primer if
  needed.
- Open documentation issue: Schema packs do not define full page templates.
  Templates must live in skills, examples, docs, or custom validation.
- Open documentation issue: End-user file URL behavior differs between
  `gbrain files signed-url` and MCP `file_url`; test chosen backend and URL
  flow before launch.
