# CLI Primer

Working notes for business technologists and brain builders.

The CLI is the trusted operating surface for a GBrain install. It is how an
operator initializes a brain, connects sources, syncs content, configures
runtime behavior, checks health, and manages remote access.

## Core Idea

The local CLI is trusted. It runs with `remote = false`.

Remote MCP callers are untrusted. They run with `remote = true`, use OAuth client
source scope, and cannot call local-only operations.

Some operations are shared between CLI and MCP. Other commands are host-side
admin tools and stay local.

In thin-client mode, the CLI routes allowed shared operations to a remote MCP
server. Local-only commands still require a local brain host.

## Setup And Verification

These commands establish or repair the install:

- `gbrain init`: creates or connects a brain, selects the database engine, sets
  provider defaults, applies migrations, and prompts for search mode choice
- `gbrain doctor`: checks health and can produce or run remediation plans
- `gbrain apply-migrations --yes`: applies schema migrations without a full
  upgrade
- `gbrain upgrade` and `gbrain post-upgrade`: update the CLI and run
  upgrade-time checks
- `gbrain migrate`: moves between supported database engines

Use `gbrain doctor` before production launch and after major upgrades.

## Configuration Commands

These commands tune how the brain runs:

- `gbrain config show|get|set|unset`: inspect or change known config keys
- `gbrain providers list|env|test`: inspect embedding provider readiness
- `gbrain models`: inspect and repair model configuration
- `gbrain search modes|stats|tune|diagnose`: inspect and tune retrieval behavior
- `gbrain storage status`: inspect storage tiers and missing files

`gbrain config set` validates known keys. It refuses live changes to embedding
model and dimensions because those choices define the existing embedding schema.

## Sources And Sync

Sources are the repo or content roots inside one brain.

Use source commands to register and manage them:

- `gbrain sources add`: register a local path or clone a remote URL
- `gbrain sources list`: see active and archived sources
- `gbrain sources current`: show which source the local resolver selected
- `gbrain sources status`: show source health, lag, embeds, failures, and queues
- `gbrain sources default`: set the brain's default source
- `gbrain sources attach`: write `.gbrain-source` for a working directory
- `gbrain sources federate|unfederate`: control cross-source default search
- `gbrain sources archive|restore|purge`: manage removal and recovery

Use sync commands to move source repo content into the brain database:

- `gbrain sync`: sync the selected source
- `gbrain sync --all`: sync all enabled local-path sources
- `gbrain sync --watch`: keep syncing on an interval
- `gbrain embed --stale`: backfill stale embeddings when needed
- `gbrain extract --stale`: backfill stale link and timeline extraction

Sync is database mutation. It reads source repos and updates indexed brain
content. Large sync runs can defer extraction or embedding work to backfill
commands and jobs.

## Capture, Import, And Files

These commands bring new material into the brain:

- `gbrain capture`: save a note or observation
- `gbrain import`: bulk-load material into the database
- `gbrain files`: manage local file upload and file storage workflows
- `gbrain put_raw_data`: store raw evidence for later extraction

File upload and file listing are local-only admin operations. They are not
exposed by HTTP MCP.

## Find And Read

These commands inspect brain state or retrieve knowledge:

- `gbrain search`: find relevant pages or chunks
- `gbrain query`: ask the brain for synthesized answers
- `gbrain think`: run a higher-level synthesis path
- `gbrain get`: read a page
- `gbrain list`: list pages
- `gbrain stats` and `gbrain health`: inspect database state
- `gbrain graph-query`: inspect graph relationships

Retrieval commands are read-oriented, but `search`, `query`, and `get` can update
retrieval telemetry such as last-retrieved time.

Remote readers see source-scoped results and filtered private material. Trusted
local callers can see the full local view.

## Schema Commands

Schema commands manage the active knowledge shape:

- `gbrain schema active`: show the resolved schema pack
- `gbrain schema list|show|validate|lint|stats|explain`: inspect packs
- `gbrain schema use`: select a pack
- `gbrain schema fork|init|edit|diff`: author or modify a pack
- `gbrain schema add-type|remove-type|update-type`: change page types
- `gbrain schema add-alias|remove-alias`: change aliases
- `gbrain schema add-link-type|remove-link-type`: change link verbs
- `gbrain schema detect|suggest|review-orphans`: find drift or missing types
- `gbrain schema sync`: apply schema-backed page updates

Schema commands can change the database and pack files. Treat schema activation
as an operating decision, not a cosmetic edit.

## Remote And Access Commands

These commands expose the brain to tools and agents:

- `gbrain serve`: run local stdio MCP
- `gbrain serve --http`: run OAuth-protected HTTP MCP
- `gbrain auth register-client`: create an OAuth client with scopes and source
  limits
- `gbrain auth list|revoke-client|test`: inspect or remove access
- `gbrain connect`: configure a remote client

Scopes are enforced by the server. `admin` implies write and read. `write`
implies read. The `agent` scope is separate.

## Jobs And Automation

These commands run background or repeated work:

- `gbrain jobs`: submit, inspect, cancel, retry, prune, and run jobs
- `gbrain autopilot`: run configured maintenance flows
- `gbrain dream`: run synthesis-oriented brain work

Use these after the source, schema, model, and search configuration are stable.

## Skills And Skillpacks

These commands work with repeatable agent workflows:

- `gbrain skillify scaffold`: create a new local skill scaffold
- `gbrain skillify check`: audit a skill
- `gbrain skillpack scaffold`: create a third-party skillpack scaffold
- `gbrain skillpack reference`: compare a host skill to an upstream skillpack
- `gbrain skillpack harvest`: copy a host skill into GBrain with privacy checks

Skill commands change repo files. They do not change the database unless the
skill itself later runs database operations.

## Mutation Map

Pure inspection commands include list, stats, health, schema show, schema lint,
provider inspection, model inspection, and search mode inspection.

Database-mutating commands include config set/unset, capture, import, sync,
embed, extract, schema sync with apply, source metadata changes, page writes,
deletes, and job submission.

Read-oriented retrieval commands include search, query, think, and get. Treat
them as reads for permissions, with the telemetry caveat above.

Workspace-mutating commands include sources attach, schema authoring commands,
skillify, skillpack scaffold or harvest, and file workflows that stage local
artifacts.

Local-only admin commands include sync, file upload/listing, extraction and
embedding workers, local doctor checks, and server hosting.

## Agent Use

An agent should resolve the brain, source, schema pack, and access scope before
running commands.

Local agents can use the local resolver. Remote agents must respect their OAuth
source scope and cannot assume the local working directory.

Agents should prefer MCP operations when running remotely and CLI commands when
operating as a trusted local caller.

## Open Documentation Issue

`docs/mcp/DEPLOY.md` still has a stale paragraph that describes file and sync
operations as remotely callable. Current HTTP MCP server code filters local-only
operations before publishing tools.
