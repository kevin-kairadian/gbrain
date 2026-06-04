# Configuration Primer

Working notes for business technologists and brain builders.

This primer explains how a GBrain install is tuned. It separates built-in
behavior, configuration, schema packs, skills, and platform code changes.

## Core Idea

Configuration tells a brain how to run.

It does not, by itself, define what knowledge means. That job belongs to the
active schema pack. It also does not create repeatable agent workflows. That job
belongs to skills.

GBrain has three practical configuration layers:

- install-level settings: where the brain lives, which database it uses, which
  providers and keys are available, and whether this CLI is a local install or a
  thin client
- brain-level settings: values stored in the brain database, such as search mode,
  the default source, active schema pack, and runtime model settings
- source-level settings: values tied to one source repo inside the brain, such as
  local path, federation, sync state, source-specific schema pack, and contextual
  retrieval override

Some repo files also affect operation. The important one for brain builders is
`gbrain.yml`, which can declare source repo storage tiers and a repo-level schema
pack.

## Database Configuration

GBrain has two database modes.

PGLite is the zero-config local default. It is the right starting point for a
single-machine brain and small source sets.

Postgres with pgvector is the production scale path. Supabase is a supported
Postgres option. Use this when the brain needs larger source sets, remote access,
or multi-machine operation.

`GBRAIN_DATABASE_URL` or `DATABASE_URL` selects Postgres and overrides the saved
database path. `GBRAIN_HOME` moves the GBrain home directory; it must be an
absolute path.

The database engine stores and searches knowledge. It does not choose the
embedding provider by itself.

## Source Configuration

A brain can contain multiple sources. A source is one repo or content root inside
the database.

Each page belongs to exactly one source. Page slugs are unique within a source,
not across the whole brain.

The default source exists for backward compatibility and is federated. New
non-default sources are isolated unless they are marked federated. An isolated
source is searched only when the caller names that source. A federated source
participates in cross-source default search.

Source selection for the trusted local CLI follows this order:

1. explicit `--source`
2. `GBRAIN_SOURCE`
3. nearest `.gbrain-source` file
4. registered source whose local path contains the current directory
5. `sources.default`
6. the only non-default source, when exactly one active non-default source exists
7. `default`

Remote MCP callers do not use that local resolver. They use the source scope in
their OAuth client.

## Schema Pack Configuration

A schema pack defines the page types, path prefixes, link verbs, aliases, and
extraction routing that make a brain readable.

Schema packs do not create source repos. They do not grant access. They do not
make agent workflows deterministic. They define the shape of knowledge.

Active schema pack resolution has several tiers:

1. trusted local per-call override
2. `GBRAIN_SCHEMA_PACK`
3. source-specific database config
4. brain-wide database config
5. `gbrain.yml`
6. file config
7. built-in fallback

Remote callers cannot force a per-call schema pack override.

Fresh `gbrain init` writes `gbrain-base-v2` for new installs unless the operator
chooses another pack. The low-level fallback is still `gbrain-base` when no
setting exists.

## Search And Retrieval

Search mode is a business decision because it controls cost, context size, and
retrieval depth.

The built-in modes are:

- `conservative`: smaller context, lower cost, no expansion
- `balanced`: the default mode, moderate context, reranking, graph signals, title
  contextual retrieval, and automatic context trimming
- `tokenmax`: largest retrieval path, expansion, larger search limit, and no
  token budget cap

`gbrain init` applies a default search mode and prints an agent-marked cost
matrix. An agent must relay that matrix and confirm the operator's choice before
continuing.

Search mode can be set as brain config. Individual `search.*` keys can override
parts of a mode bundle. Trusted local callers can pass a per-call mode. Remote
callers use the configured mode and cannot force a per-call mode.

## Model And Provider Settings

GBrain separates model configuration from database configuration.

During init, explicit provider flags win. If no explicit flags are passed, init
checks environment variables for ready providers. If exactly one provider is
ready, init can persist that choice. If several are ready in an interactive
terminal, init asks. In non-interactive mode, ambiguous provider selection fails
loudly.

Local embedding providers are not auto-selected. They must be named explicitly.

Embedding model and embedding dimensions are schema-sizing choices for an
existing brain. `gbrain config set` refuses to change them. To switch them for an
existing brain, use the supported reinit or migration path for the database
engine.

Runtime model settings, provider base URLs, chat model, expansion model, and
many search settings can be configured through environment variables, file
config, or database config.

## Storage Settings

Source repo storage tiers live in `gbrain.yml` under `storage`.

The canonical keys are:

- `db_tracked`: files tracked in the source repo and database
- `db_only`: files stored in the database/storage layer but not meant for source
  repo tracking

Older aliases are read with warnings, but new docs and new configs should use
the canonical keys.

File upload storage can use local storage, S3, or Supabase storage. On PGLite,
storage tiering has limited effect because pages live in the local database
file. Full tiering matters most with Postgres or Supabase.

## Environment Variables Vs Config

Environment variables are runtime operator overrides. They win over saved config
for the keys GBrain reads from the environment.

File config lives under the GBrain home directory and is machine-local.

Database config is stored in the brain and is the right place for runtime brain
settings such as search mode, default source, and brain-wide schema pack.

Do not save a one-off environment override as config unless it should become the
brain's operating rule.

## Configure Once

Decide these before production use:

- database mode: PGLite, Postgres, or Supabase
- access boundary: one brain or separate brains
- sources: which repos belong in the brain and which are federated
- embedding provider, embedding model, and dimensions
- initial schema pack and source-specific schema pack overrides
- search mode and cost posture
- raw file and evidence storage
- model/provider keys and base URLs
- remote MCP publishing and OAuth client scopes
- whether remote clients can read the skill catalog

## Monitor Over Time

Monitor these after launch:

- `gbrain doctor`
- `gbrain sources status`
- sync freshness and embedding coverage
- search mode stats and tuning output
- model/provider health
- storage status
- OAuth clients and scopes
- schema pack drift and orphan review

## Open Documentation Issues

The current architecture docs and older KT_STUFF schema primer still describe
`gbrain-base` as the default schema pack. Current init code writes
`gbrain-base-v2` for fresh installs, while the resolver fallback remains
`gbrain-base`.

`docs/mcp/DEPLOY.md` contains a stale operations paragraph that says remote MCP
can call file and sync operations. Current server code filters local-only
operations out of HTTP MCP.
