# Access and Authorization

Working notes for understanding access boundaries in GBrain.

## The Main Boundaries

**Brain**
A brain is the database and access boundary. Use separate brains when data ownership, customer isolation, or access policy is different.

**Source Repo**
A source repo is a repository of markdown pages inside a brain. It is the primary record for those pages. Source repos separate content organization inside a brain, but they are not the same boundary as a separate brain.

**Database**
The database stores derived indexes, chunks, links, facts, takes, timeline entries, file metadata, raw JSON sidecars, jobs, logs, tokens, and configuration.

**Object Storage**
Large files and media can be stored outside git with pointer files or file metadata, depending on storage configuration.

## Local CLI Callers

The local CLI is trusted.

Local CLI operations run with `remote = false`. They can use local filesystem paths and can access local-only operations such as file upload and file management.

Local writes can preserve caller-provided provenance fields. Local page writes can also run automatic link and timeline extraction when those features are enabled.

## Remote and MCP Callers

Remote and MCP callers are untrusted by default.

Remote operations run with `remote = true`. Sensitive code treats anything other than explicit `remote = false` as untrusted.

Remote page reads strip the takes fence and filter private facts. Facts marked `world` can remain visible.

Remote page writes ignore caller-provided provenance and stamp the page as an MCP write. Remote writes also strip gate-owned frontmatter markers. Automatic link and timeline extraction is skipped for ordinary remote writes because untrusted text must not be allowed to create graph edges through prompt-injected content.

Trusted workspace subagent paths can re-enable some automatic behavior when GBrain has explicit allowed slug prefixes.

## HTTP MCP Authorization

The HTTP MCP server uses bearer-token authorization.

Local-only operations are not exposed over HTTP MCP.

Scopes control access:

- `read` allows read operations.
- `write` implies `read`.
- `admin` implies `write`, `read`, `sources_admin`, and `users_admin`.
- `sources_admin` and `users_admin` are separate administrative scopes.
- `agent` is its own scope. It is not implied by `admin`.

OAuth clients can be registered with a write source and a federated read-source list. For remote HTTP calls, source routing comes from the authenticated client context, not from the local CLI resolver.

## Source Routing

For local CLI use, GBrain can resolve the active source from command options, environment, `.gbrain-source`, registered local paths, source defaults, or the `default` source.

Remote callers do not inherit that local resolver.

Read operations use the caller's allowed source list when present. If no allowed source list is present, they use the caller's current source. This keeps federated reads explicit.

Write operations use the caller's write source.

## Raw Evidence, Files, and Provenance

Pages have provenance fields such as source kind, source URI, ingestion path, and ingestion time.

Local callers can provide provenance on trusted writes. Remote MCP page writes are server-stamped.

Raw JSON API responses can be stored as raw data sidecars for a page and source. These are database records, not markdown page sections.

File operations are local-only administrative operations. The local CLI can upload, verify, mirror, restore, and clean files. Large raw files or media can be stored in object storage with redirect pointers.

Sync skips raw-file folders such as `.raw` and does not treat them as normal markdown pages.

## Source Control and Storage Choices

Use GitHub Enterprise or another private git system for source repos when the markdown knowledge belongs in version control.

Use object storage for large files, media, and artifacts that do not belong directly in git.

Use database-only storage only when the operating model accepts that the data is not represented as normal markdown pages in the source repo.

Do not assume a fact is safe because an agent cannot read it. If private information is committed to a source repo, it still exists in git history and must be governed there.

## Customer Isolation Rule

Use separate brains for separate customers or separate legal access boundaries.

Use sources inside one brain only when those sources can safely share the same database boundary and access model.

Use federated source reads intentionally. They are useful when an authorized agent needs to search across selected sources, but they are not a replacement for brain-level isolation.
