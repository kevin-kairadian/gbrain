# Schema Pack Primer

Working notes for understanding what schema packs do.

## What a Schema Pack Is

A schema pack is the rule set GBrain uses to interpret a source repo.

It tells GBrain which page types exist, which folder paths map to those types, what aliases the types have, which link rules are available, which types can participate in expert routing, and which types are declared extractable.

For one source repo, GBrain resolves one active schema pack at a time.

## Where Packs Live

GBrain ships bundled schema packs. It also loads user packs from `~/.gbrain/schema-packs/<pack-name>/pack.yaml`, `pack.yml`, or `pack.json`.

A pack can extend another pack. A pack can also borrow selected page types or link types from another pack.

## How GBrain Chooses a Pack

GBrain resolves the active schema pack in this order:

1. A per-call schema-pack option from a trusted local CLI caller.
2. `GBRAIN_SCHEMA_PACK`.
3. A per-source database setting.
4. A brain-wide database setting.
5. `gbrain.yml`.
6. `~/.gbrain/config.json`.
7. `gbrain-base`.

Remote and MCP callers cannot override the schema pack per call.

## Main Pack Fields

**Page types**
Named types such as `person`, `company`, `meeting`, or a domain-specific type. The `name` field is the value that appears in page frontmatter as `type`.

**Path prefixes**
Folder rules used to infer page type from file path. If multiple prefixes overlap, declaration order matters because type inference is first match wins.

**Aliases**
Alternate names for a page type. Aliases help query and routing behavior treat related type names as connected.

**Link types**
Named relationship verbs. Pack link rules can add page-type-bound rules and regex-backed rules. Built-in link inference still exists as a fallback.

**Frontmatter links**
Manifest entries for frontmatter-field relationships. GBrain validates these entries and includes them in the schema graph. The current built-in frontmatter link extractor uses code-level mappings, so verify runtime support before relying on pack-specific frontmatter links to create links.

**Extractability**
A declaration that a type is intended to participate in extraction tooling. This can be a boolean or a structured extractability spec with prompt, fixture, and benchmark settings.

Important: the current hot facts backstop still has a code-level eligibility list. Do not assume a new page type will get facts extracted only because its pack says `extractable: true`. Verify the extraction path or extend GBrain.

**Expert routing**
A flag that lets expert-finding operations include that page type.

## Deterministic Type Matching

The page frontmatter `type` matches `page_types[].name`.

```yaml
page_types:
  - name: customer
    primitive: entity
    path_prefixes:
      - customers/
```

```markdown
---
title: Acme Corp
type: customer
---
```

The folder path can infer the same type:

```text
customers/acme.md maps to customer
```

For deterministic operation, use both:

- File the page under one of the type's `path_prefixes`.
- Set page frontmatter `type` to the matching `page_types[].name`.
- Keep one business meaning per type name.
- Keep one canonical folder prefix per type when possible.

Subfolders are allowed. Put the more specific prefix before the broader prefix:

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
customers/enterprise/acme.md maps to enterprise_customer
customers/acme.md maps to customer
```

This removes model judgment from filing. The agent reads the active schema pack, chooses the matching prefix, and writes the matching `type`.

## What Packs Do

Schema packs do these jobs:

- Infer page type from source path.
- Define the vocabulary of page types and relationship verbs.
- Provide aliases for query and routing behavior.
- Mark which types are available for expert routing.
- Declare extractability intent and extraction benchmark metadata.
- Expose frontmatter-link declarations to schema linting and graph views.
- Let different source repos inside one brain use different interpretation rules.
- Let schema sync backfill missing `page.type` values for rows that match pack prefixes.

## What Packs Do Not Do

Schema packs do not store knowledge.

They do not create database tables.

They do not replace the source repo.

They do not define a complete markdown page template.

They do not make every custom frontmatter field structured.

They do not replace GBrain's built-in facts, takes, link, tag, and timeline extractors.

## Starter Packs and Real Packs

Starter packs are useful for common page shapes and for learning the mechanism.

Serious domain use needs a domain-specific pack. A mining brain, legal brain, customer-success brain, or investment brain needs deterministic page types, folder mappings, and relationship names that match the business domain.

If the business model requires a new kind of structured row, workflow, or extraction behavior, that is beyond schema-pack configuration. It requires extending GBrain itself.
