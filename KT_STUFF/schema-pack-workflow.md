# Schema Pack Workflow

Working notes for creating a real schema pack.

## Goal

A schema pack makes a source repo legible to GBrain.

It does not describe every possible page. It defines the page types, folder rules, aliases, relationships, and extraction intent that GBrain needs to classify, search, route, and audit the source.

## Step 1: Define the Business Purpose

Start with the business questions the brain must answer.

Examples:

- Which customers are at renewal risk?
- Which projects depend on which permits?
- Which founders changed metrics over time?
- Which internal experts know a topic?

Do not start by copying a starter pack. Start with the decisions and questions the brain must support.

## Step 2: Choose Brain and Source Boundaries

Use a separate brain when the data owner, access policy, or customer boundary is different.

Use separate source repos inside a brain when the knowledge belongs in the same access boundary but comes from different repositories or operating areas.

Each source can have its own schema pack when its folder structure or domain vocabulary is different.

## Step 3: Inventory the Source Repo

List the real folders and page shapes.

For each repeated shape, decide whether it deserves a page type. Good page types represent durable business concepts, not one-off sections.

Examples:

- `customers/` maps to `customer`
- `projects/` maps to `project`
- `contracts/` maps to `contract`
- `meetings/` maps to `meeting`

If a folder contains mixed content, split the folder or use a broader type. Do not rely on ambiguous path rules.

## Step 4: Define Page Types

For each type, define:

- Name.
- Primitive.
- Path prefixes.
- Aliases.
- Whether it is extractable.
- Whether it participates in expert routing.

Keep names short and stable. Use aliases for alternate business language.

## Step 5: Map Folder Prefixes

Path prefixes drive type inference.

Each page type's `name` is the value pages use in frontmatter as `type`.

Map each important type to a stable folder or subfolder:

```yaml
page_types:
  - name: enterprise_customer
    path_prefixes:
      - customers/enterprise/

  - name: customer
    path_prefixes:
      - customers/
```

Then instruct agents to write:

```markdown
---
title: Acme Corp
type: enterprise_customer
---
```

at:

```text
customers/enterprise/acme.md
```

Avoid duplicate prefixes. Avoid overlapping prefixes unless the more specific prefix is declared before the broader one.

For deterministic operation, use explicit frontmatter `type` and a matching path prefix. Treat path inference as a safety net, not the normal authoring path.

Run schema lint before relying on the pack. GBrain has lint rules for duplicate aliases, undeclared link references, duplicate prefixes, overlapping prefixes, expert-routing types without prefixes, and other pack mistakes.

## Step 6: Define Relationships

Decide which relationships matter to the business.

Use built-in links when normal markdown links, wiki links, or built-in frontmatter mappings are enough.

Use schema-pack link types when the domain needs named relationships such as `depends_on`, `regulated_by`, `owns`, or `supersedes`.

Pack link rules can add deterministic page-type and regex inference. They do not remove every built-in link behavior.

Schema packs can declare frontmatter-link rules for linting and schema graph views. The current runtime frontmatter link extractor is code-level. If a new frontmatter field must create links, verify support in the current code or extend GBrain.

If the relationship requires a new extractor, add code to GBrain.

## Step 7: Decide What Is Extractable

Only mark a type extractable when there is a reason to extract claims from it.

For simple structured claims, use GBrain's built-in facts and takes fences.

For domain-specific extraction, create the prompt and fixture corpus, then benchmark it. GBrain has schema-pack support for extractability specs, scaffolded prompts, fixtures, and extraction benchmarks.

Before depending on extraction for a new page type, verify the current extraction path. Some extraction paths are built-in cycle phases, and the hot facts backstop still uses a code-level eligibility rule.

## Step 8: Build and Validate the Pack

Use the schema authoring tools to create or fork a pack, then edit the manifest.

Check the pack with:

- `gbrain schema validate`
- `gbrain schema lint`
- `gbrain schema stats`
- `gbrain schema review-orphans`
- `gbrain schema sync` before `gbrain schema sync --apply`

Schema sync backfills missing page types in the database for pages whose source paths match pack prefixes. It does not rewrite every page layout.

## Step 9: Test with Real Pages

Use a small set of representative pages first.

Verify:

- The right page types are inferred.
- Search returns the expected pages.
- Links point to the right targets.
- Facts, takes, and timeline entries appear where expected.
- Expert routing includes only the intended types.
- Remote agents see only what their access allows.

Then sync the larger source repo.

## Step 10: Operate Carefully

Version the pack.

Review changes before applying them to a production brain.

Commit pack files to source control when they matter to a team.

Use starter packs for learning. Use domain-specific packs for serious business use.
