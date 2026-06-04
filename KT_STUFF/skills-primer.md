# Skills Primer

Working notes for business technologists and brain builders.

Skills are repeatable agent workflows. They tell an agent what to do, what to
check, and what output to produce.

## Core Idea

A skill is an instruction file, not a security boundary.

The normal shape is:

- `skills/<skill-name>/SKILL.md`
- frontmatter with name, description, triggers, tools, mutating behavior, and
  write targets
- body text that defines the contract, phases, checks, and outputs
- optional scripts or tests for mechanical work

The trigger index reads skill frontmatter. Resolver docs can add routing
guidance, but frontmatter triggers are the canonical trigger source.

## Skills Vs Schema Packs

Schema packs define the structure of knowledge.

They define page types, path prefixes, aliases, link verbs, extraction rules,
and expert routing.

Skills define repeatable work.

They define how an agent should gather evidence, decide what matters, use
commands or MCP operations, write pages, verify output, and report results.

Configuration selects active sources, search mode, model settings, storage, and
schema pack choices.

Platform code enforces database behavior, operation permissions, local-only
rules, extraction pipelines, provider support, and server behavior.

## When To Create A Skill

Create a custom skill when the work is repeated and judgment-heavy.

Good skill candidates have:

- a clear trigger phrase
- a named output
- steps that must happen in order
- evidence or verification requirements
- a clear rule for which pages or files may be changed
- enough complexity that a short prompt would drift over time

Examples:

- mining assay ingestion
- project review
- evidence verification
- diligence memo preparation
- customer update workflow

Do not create a skill just to rename a page type, add a link verb, change search
mode, or switch model providers.

## What Requires What

Use configuration when the operator needs to tune runtime behavior: database,
source, search, model, provider, storage, access, or schema pack selection.

Use a schema pack when the brain needs a new knowledge structure: page type,
path prefix, alias, link verb, extraction target, or expert routing rule.

Use a skill when the brain needs a repeatable agent process over existing or
planned knowledge structures.

Use platform code changes when the product must enforce new behavior: database
schema, operation permissions, remote/local trust rules, provider integration,
server API behavior, or extraction engine logic.

## Tools In Skills

The `tools` field declares the surfaces a skill expects an agent to use. It does
not grant permission by itself.

Actual permission comes from the operating surface:

- trusted local CLI context
- remote MCP scopes
- source scope
- local-only operation rules
- filesystem and repo access

Scripts inside a skill should do deterministic mechanical work. The skill body
should define judgment, ordering, verification, and reporting rules.

## Scaffold And Check

`gbrain skillify scaffold` creates a local skill scaffold. It requires a
description and writes the skill file, starter script, routing eval, test, and
resolver row.

`gbrain skillify check` audits a skill.

`gbrain skillpack scaffold` creates a third-party skillpack scaffold. Skillpacks
are distributed scaffolds. When a host repo adopts one, the host repo owns the
copied files.

`gbrain skillpack harvest` copies a host skill into GBrain with genericization
and privacy checks. It does not publish automatically.

Skillpack bootstrap steps are displayed to the operator. They are not executed
automatically by the scaffold flow.

## Remote Skill Catalog

GBrain can publish skill docs through MCP when `mcp.publish_skills` is enabled.

Remote clients can list and read exposed skills through `list_skills` and
`get_skill`. The catalog is read-only. It does not install skills into the
client and does not bypass scopes.

## Respect The Brain Model

A skill should name how it handles:

- brain selection
- source selection
- schema pack assumptions
- writes to pages or files
- private material
- evidence requirements
- verification steps
- remote versus local operation

Remote agents must follow their OAuth source scope. Local agents can use the
local source resolver. Local-only operations, such as sync and file upload, stay
on the trusted host.

## Examples

Mining assay ingestion:

- schema pack: sample, assay, project, company page types and link verbs
- skill: read assay files, normalize measurements, cite evidence, update project
  pages, and flag missing units
- platform code change: only needed if GBrain must enforce a new assay database
  table or extractor

Diligence memo:

- schema pack: company, person, fund, deal, risk, metric types
- skill: gather evidence, compare claims, write a memo, and verify citations
- configuration: choose source scope, search mode, provider, and model budget

Customer update workflow:

- schema pack: customer, meeting, request, decision, follow-up types
- skill: summarize new interactions, update customer pages, and produce an
  outbound update
- platform code change: only needed for a new integration or enforced server
  behavior
