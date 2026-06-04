# Page Model

Working notes for designing and writing GBrain pages.

This is for a business technologist creating a company-specific GBrain schema pack. It explains what every GBrain brain already knows how to read, what a schema pack can configure, and what you must document separately so agents know how to write good pages.

## Core Idea

A GBrain page is one markdown file in a source repo.

The markdown file is the primary record. The database stores the parsed and indexed version so GBrain can search, chunk, link, extract facts, store timelines, and serve agents.

A schema pack helps GBrain classify and route pages. It does not define the full page layout.

That means page design has two parts:

1. Configure the schema pack so GBrain knows the page type and path.
2. Write page-authoring instructions so humans and agents know what to put in that type of page.

## Native Page Shape

Every GBrain page uses the same basic shape:

```markdown
---
title: Acme Renewal Risk
type: customer
tags: [renewal, risk]
owner: people/alice-example
status: at_risk
---

# Acme Renewal Risk

Short durable summary.

## Current State

What is true now, with sources.

## Open Questions

What still needs to be checked.

<!-- timeline -->

## Timeline

- **2026-05-14** | Customer raised renewal concern. [Source: Meeting "Q2 renewal", 2026-05-14]
```

The frontmatter block is YAML. The body is markdown.

GBrain treats these frontmatter fields specially:

- `title`: display title. If missing, GBrain can infer a title from the filename.
- `type`: page type. If missing, GBrain can infer it from the active schema pack's path prefixes. If no schema-pack path rule matches, the fallback type is `concept`.
- `tags`: indexed tags.
- `slug`: optional. The file path is the authority. If `slug` is present during file import, it must match the path-derived slug.

Other frontmatter fields are preserved. They do not automatically become first-class database fields.

## Type Matching Rule

Page frontmatter `type` must match the `name` of a `page_types` entry in the active schema pack.

Schema-pack entry:

```yaml
page_types:
  - name: customer
    primitive: entity
    path_prefixes:
      - customers/
```

Page:

```markdown
---
title: Acme Corp
type: customer
---
```

Path:

```text
customers/acme.md
```

The deterministic match is:

```text
page type: customer
schema page_types[].name: customer
path_prefixes: customers/
```

If `type` is present, GBrain uses it. If `type` is missing, GBrain can infer it from the file path and the active schema pack's `path_prefixes`.

For company use, do not rely on inference as the authoring norm. Set `type` explicitly and place the file under the matching prefix. This gives humans, agents, sync, and schema lint the same answer.

Subfolders can represent more specific types:

```yaml
page_types:
  - name: enterprise_customer
    primitive: entity
    path_prefixes:
      - customers/enterprise/

  - name: customer
    primitive: entity
    path_prefixes:
      - customers/
```

```text
customers/enterprise/acme.md
type: enterprise_customer
```

When prefixes overlap, put the specific subfolder entry before the broad folder entry in the schema pack.

## Native Structures

These structures are native to GBrain. They work across all brains.

| Structure | Where It Lives | What GBrain Does |
|---|---|---|
| Frontmatter | YAML block | Preserves custom keys; treats `title`, `type`, `tags`, and `slug` specially. |
| Body | Markdown | Stores as compiled truth and chunks for search. |
| Tags | Frontmatter | Adds tags to the tag index. |
| Links | Markdown links, wiki links, selected frontmatter fields | Extracts relationship edges. |
| Aliases | `aliases` frontmatter | Projects aliases into the alias index. |
| Timeline | Timeline marker plus dated lines | Extracts timeline entries from strict `YYYY-MM-DD` rows. |
| Facts | GBrain facts fence | Derives rows in the facts table. |
| Takes | GBrain takes fence | Derives rows in the takes table. |
| Raw evidence | Files, raw JSON sidecars, source links | Preserves provenance when ingestion code stores it. |

Use these before inventing a custom structure.

## Body Sections Are Authoring Conventions

Markdown headings such as `## Current State`, `## Decision Log`, or `## Risks` are useful. GBrain preserves and indexes them as body content.

They are not automatically structured.

For example, a `## Renewal Risks` section is searchable text. It does not become a `renewal_risks` table. To make it database-native, use an existing GBrain structure or extend GBrain code.

## Facts

Use facts for structured claims about an entity.

Facts are best for things that can be true, false, dated, superseded, or used in trajectory analysis.

```markdown
## Facts

<!--- gbrain:facts:begin -->
| # | claim | kind | confidence | visibility | notability | valid_from | valid_until | source | context |
|---|-------|------|------------|------------|------------|------------|-------------|--------|---------|
| 1 | Contract renewal is due in Q4 | fact | 0.9 | private | high | 2026-05-14 |  | meeting/q2-renewal |  |
| 2 | Reported 12 open support tickets | event | 0.85 | private | medium | 2026-05-14 |  | support export |  |
<!--- gbrain:facts:end -->
```

Use `visibility: world` only for facts that are safe for remote readers. Private facts are filtered from remote page reads.

## Takes

Use takes for beliefs, judgments, bets, and analysis.

The `who` column means who holds the belief, not who the belief is about.

```markdown
## Takes

<!--- gbrain:takes:begin -->
| # | claim | kind | who | weight | since | source |
|---|-------|------|-----|--------|-------|--------|
| 1 | Renewal risk is mainly product reliability, not price | take | brain | 0.75 | 2026-05-14 | compiled from meeting/q2-renewal |
| 2 | Customer champion believes rollout can recover by September | belief | people/alice-example | 0.7 | 2026-05-14 | meeting/q2-renewal |
<!--- gbrain:takes:end -->
```

Takes are append-only. Supersede old rows instead of rewriting history.

## Timeline

Use timeline for dated events.

GBrain recognizes dated lines like this:

```markdown
<!-- timeline -->

## Timeline

- **2026-04-01** | Contract signed. [Source: CRM export, 2026-04-02]
- **2026-05-14** | Renewal risk raised in QBR. [Source: Meeting "Q2 renewal", 2026-05-14]
```

The date must be a real `YYYY-MM-DD` date. Details can follow as continuation text under the dated line.

## Links

Use deterministic links to connect pages.

```markdown
Acme is owned by [Acme Corp](companies/acme-corp.md).
The renewal depends on [Reliability Program](projects/reliability-program.md).
```

Do not ask the model to invent links. Build links from known slugs, resolver results, API responses, or existing source data.

GBrain can extract links from markdown links, wiki links, and selected built-in frontmatter mappings. Schema-pack link types can add named relationship verbs and some deterministic inference rules, but they do not replace every built-in extractor.

## Custom Fields

Custom frontmatter is useful for stable metadata:

```yaml
crm_id: "cust_123"
status: at_risk
owner: people/alice-example
renewal_date: 2026-10-31
plan: enterprise
```

These fields are preserved. They are good for humans, agents, and future code.

Do not treat them as database-native unless GBrain code already indexes them that way. If the business needs a reliable database filter, workflow, or report over a custom field, that is a GBrain extension, not just a schema-pack setting.

## What the Schema Pack Controls

A schema pack can define:

- Page type names.
- Path prefixes that map files to types.
- Type aliases.
- Link type names and some deterministic link inference rules.
- Frontmatter-link declarations for schema linting and graph views.
- Extractability intent and extraction benchmark metadata.
- Expert-routing flags.
- Per-source interpretation rules.
- Schema sync rules that backfill missing `page.type` values from paths.

For example:

```yaml
page_types:
  - name: enterprise_customer
    primitive: entity
    path_prefixes:
      - customers/enterprise/
    aliases:
      - strategic_account
    extractable: true
    expert_routing: true

  - name: customer
    primitive: entity
    path_prefixes:
      - customers/
    aliases:
      - account
    extractable: true
    expert_routing: true

  - name: contract
    primitive: annotation
    path_prefixes:
      - contracts/
    aliases:
      - agreement
    extractable: true
    expert_routing: false
```

This tells GBrain that `customers/enterprise/acme.md` is an `enterprise_customer` page, `customers/acme.md` is a `customer` page, and `contracts/acme-msa.md` is a `contract` page.

It does not tell the model what sections a customer page must contain.

## What the Schema Pack Does Not Control

A schema pack does not define:

- Required markdown sections.
- Required frontmatter keys beyond the page type and path behavior.
- A page template.
- A validation schema for custom frontmatter.
- New database tables.
- New extraction code.
- A complete prompt for how an agent writes the page.

The schema-pack manifest is strict. Adding an informal `template:` field to a page type is not supported unless GBrain extends the manifest schema.

## How to Design a Custom Page Type

For each custom page type, define two artifacts.

**1. Schema-pack entry**

This is the machine-readable classification rule.

Include:

- `name`
- `primitive`
- `path_prefixes`
- `aliases`
- `extractable`
- `expert_routing`
- link types if this type needs named relationships

**2. Page authoring guide**

This is the human and agent instruction.

Include:

- Canonical path pattern.
- Required frontmatter fields for your organization.
- Recommended body sections.
- Which native structures to use.
- Example page.
- Anti-patterns.

GBrain has the first artifact natively. You must create the second artifact today as a skill, convention doc, team README, or examples.

## Page Type Authoring Template

Use this format when documenting a company-specific type:

```markdown
# Page Type: customer

## Purpose

One page per customer account. Use this page for durable account context, renewal state, key relationships, and dated customer events.

## Schema-Pack Config

- type: customer
- primitive: entity
- path prefix: customers/
- aliases: account
- extractable: true
- expert routing: true

## Required Frontmatter

- title
- type: customer
- status
- owner
- crm_id
- renewal_date

## Recommended Sections

- Summary
- Current State
- Stakeholders
- Risks
- Open Questions
- Facts
- Takes
- Timeline
- See Also

## Native Structures To Use

- Use links for related people, companies, projects, contracts, and meetings.
- Use facts for dated customer claims and metrics.
- Use takes for account judgment and confidence-weighted analysis.
- Use timeline for customer events.
- Preserve source files with raw evidence when applicable.

## Do Not

- Do not put raw transcripts directly in the customer page.
- Do not create a customer page for a one-off lead unless it passes the notability gate.
- Do not invent links or CRM identifiers.
- Do not rely on custom fields as database-native filters unless code supports them.
```

## Example: Customer Page

```markdown
---
title: Acme Corp
type: customer
tags: [customer, renewal]
status: at_risk
owner: people/alice-example
crm_id: "cust_123"
renewal_date: 2026-10-31
aliases: [Acme]
---

# Acme Corp

Acme Corp is an enterprise customer with a Q4 renewal. The current risk is product reliability, not budget. [Source: Meeting "Q2 renewal", 2026-05-14]

## Current State

- Renewal owner: [Alice Example](people/alice-example.md)
- Related project: [Reliability Program](projects/reliability-program.md)
- Related contract: [Acme MSA](contracts/acme-msa.md)

## Risks

- Reliability incidents are blocking executive confidence.
- Support backlog is visible to the customer champion.

## Open Questions

- Which reliability incidents must be closed before renewal?
- Does procurement require a new security review?

## Facts

<!--- gbrain:facts:begin -->
| # | claim | kind | confidence | visibility | notability | valid_from | valid_until | source | context |
|---|-------|------|------------|------------|------------|------------|-------------|--------|---------|
| 1 | Renewal date is 2026-10-31 | fact | 0.95 | private | high | 2026-05-14 |  | crm export |  |
| 2 | Customer reported 12 open support tickets | event | 0.85 | private | medium | 2026-05-14 |  | support export |  |
<!--- gbrain:facts:end -->

## Takes

<!--- gbrain:takes:begin -->
| # | claim | kind | who | weight | since | source |
|---|-------|------|-----|--------|-------|--------|
| 1 | Renewal risk is recoverable if reliability work ships by September | take | brain | 0.7 | 2026-05-14 | compiled from meeting/q2-renewal |
<!--- gbrain:takes:end -->

<!-- timeline -->

## Timeline

- **2026-05-14** | Renewal risk raised in QBR. [Source: Meeting "Q2 renewal", 2026-05-14]
- **2026-06-02** | Reliability Program linked as recovery plan. [Source: Project review, 2026-06-02]

## See Also

- [Acme MSA](contracts/acme-msa.md)
- [Reliability Program](projects/reliability-program.md)
```

## What to Model in the Schema Pack

Put something in the schema pack when it changes how GBrain classifies, routes, searches, or extracts.

Good schema-pack candidates:

- A repeated business object with its own folder.
- A type that needs expert routing.
- A type that needs extraction.
- A type that needs aliases.
- A relationship that needs a named link verb.
- A source repo that needs different interpretation rules.

Do not add a schema type for a one-off section or a small temporary folder. Use the nearest existing type plus frontmatter or tags.

## What to Put in Page Instructions

Put something in a page authoring guide when it tells the model how to write.

Good page-instruction candidates:

- Required sections.
- Required business metadata.
- Citation standards.
- How much summary versus raw evidence to include.
- Which related pages to link.
- Which facts or takes to write.
- Examples of good and bad pages.

This is the missing layer for most custom company packs. A pack alone gives the model the noun. The authoring guide tells the model how to write the page.

## What Requires GBrain Extension

Extend GBrain itself when the company needs behavior that is not just classification or authoring guidance.

Examples:

- A new database-native table.
- A new extractor for a custom section.
- A custom frontmatter field that must be queryable as a first-class filter.
- A runtime validator that rejects pages missing required sections.
- A new remote-access visibility rule.
- A custom report built from page fields.

Until then, custom fields and custom sections are preserved and searchable, but not database-native.

## Adoption Checklist

For each company-specific type:

1. Add the type to the schema pack.
2. Choose a stable folder prefix.
3. Decide aliases.
4. Decide whether it is extractable.
5. Decide whether it participates in expert routing.
6. Define relationship verbs if needed.
7. Write a page authoring guide.
8. Include one complete example page.
9. Add the authoring guide to the relevant skill or agent instructions.
10. Test with sample pages, then run schema lint, schema stats, schema sync, and search verification.
