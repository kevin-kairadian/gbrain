# Brain Primer

Working notes for understanding GBrain from the ground up.

## What is GBrain

GBrain is a system for storing knowledge in an agent-friendly, version-controlled format so AI systems can search it, update it, and trace claims back to their sources. The core idea is to capture knowledge from wherever it originates and store it in a way that is useful for AI and humans.

The primary container GBrain uses to store knowledge is a "brain." A brain can preserve links back to the original evidence when applicable. One GBrain installation can support multiple brains, depending on the compute and storage available.

## What is a Brain

A brain is a separate knowledge container. It is the unit GBrain searches, updates, secures, and routes agents into.

A brain is made up of:

**Source Repos**
Git repositories of markdown pages that hold the brain's knowledge. Source repos are the primary record: if the database is lost, GBrain can rebuild it from the source repos. GBrain uses schema packs to understand how each source repo is organized.

**Raw Evidence and Provenance**
Supporting evidence such as `.raw/` files, uploaded documents, pointer files, source pages, or links to external systems. This is how claims can be traced back to where they came from when evidence exists.

**Database**
A PGLite or Postgres database built from the source repos. It stores indexes, links, embeddings, facts, timelines, and other derived data so agents can retrieve relevant information quickly.

One GBrain installation can support many brains, but each brain has its own content, settings, access rules, and database boundary.

## What is a Schema Pack

A schema pack is the rule set GBrain uses to interpret a source repo.

It tells GBrain what page types exist, which folder paths map to each type, and how agents should treat those types. For example, a personal brain might have people, projects, trips, and daily notes. A mining brain might have projects, companies, deposits, assays, permits, reports, and claims.

The source repo stores the knowledge in markdown pages. The schema pack tells GBrain how to classify, search, extract from, and route those pages.

A brain can have multiple source repos. By default, those sources can use the same brain-level schema pack. When needed, a specific source repo can override that with its own schema pack. For any one source repo, GBrain resolves one active schema pack at a time.

## What is a Page

A page is an individual markdown file in a source repo. It is the basic unit of knowledge GBrain reads, writes, indexes, and retrieves.

A valid GBrain page uses frontmatter at the top and body content below it. The frontmatter identifies the page with fields such as `title`, `type`, and `created`. The body holds the actual knowledge: notes, summaries, facts, timelines, links, decisions, or other domain-specific content.

A page can be simple prose, or it can include structured sections that GBrain knows how to extract into the database. The page remains the source of truth; the database stores the indexed version so agents can find and use it quickly.

GBrain has several built-in page structures it knows how to read: frontmatter fields, tags, markdown links, wiki links, facts, takes, and timeline sections. These are optional. A page does not need all of them.

You can add your own frontmatter fields or markdown sections without defining them first. GBrain will preserve them, and body content can still be indexed and made available to agents when the page is retrieved. But custom fields or sections do not automatically become new database tables or structured rows. For database-structured knowledge, you use GBrain's built-in structures or extend GBrain itself.

A schema pack does not define a required page layout. It defines how pages are classified and routed, and whether certain page types participate in extraction.

## Key Rules

The source repos are the primary record. The database is rebuilt from them.

Schema packs classify and route pages. They do not store knowledge or create new database tables.

Pages can contain normal markdown, built-in structured sections, and custom fields or sections. Custom fields and sections are preserved and searchable, but they are not automatically database-structured.

Raw evidence should be preserved when useful, but GBrain only owns what is stored in the brain or linked from it.

## Structured Business Systems

GBrain does not replace line-of-business systems or structured databases. If the source of truth is backed by a traditional DBMS, that system should remain the source of truth. GBrain stores descriptions, schema notes, data dictionaries, query examples, extracted insights, and decisions that help agents understand and use that structured data, including querying it when authorized.
