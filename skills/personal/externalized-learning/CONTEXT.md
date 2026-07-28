# Context: Externalized Learning

## Glossary

- **Externalized Knowledge**: Structured, persistent knowledge extracted by the agent from conversations and external sources. Consumed by the agent in future sessions via progressive disclosure. Analogous to Karpathy's LLM wiki pattern — the agent builds its own long-term memory.
- **Skill (Externalized Learning)**: An agent skill that manages the creation, import, maintenance, and consumption of Externalized Knowledge.
- **Knowledge Entry**: A single markdown file representing one atomic unit of Externalized Knowledge. The primary format for agent consumption.
- **Knowledge Index**: A JSON file with four sections: `entries` (canonical registry keyed by slug), `tags` (inverted index: tag → entry slugs), `keywords_index` (inverted index: keyword → entry slugs for fuzzy lookup), and `usage` (per-entry stats: `created`, `last_updated`, `last_accessed`, `access_count`). Used by the agent for navigation, lookup, and maintenance decisions.
- **Topic**: A coarse grouping of Knowledge Entries by domain (e.g., `auth/`, `networking/`, `database/`). Represented as a folder in the knowledge directory.
- **Tag**: A fine-grained cross-cutting label in the Knowledge Index (e.g., `#glossary`, `#pattern`, `#decision`, `#how-to`). An entry can have multiple tags and belong to exactly one Topic.
- **Entry Anatomy**: Each Knowledge Entry is a markdown file with: a canonical title, a tags list, an auto-maintained last-updated timestamp, explicit cross-references to related entries (`[[other-entry]]`), and a free-form body. Loose conventions for body sub-headings (e.g., `## Context`, `## Key Points`, `## Gotchas`) but no enforced schema.

## Core Behaviors

1. **Auto-extraction**: The agent opportunistically extracts definable, reusable knowledge during conversation (domain terms, discovered patterns, non-obvious config quirks, resolved decisions).
2. **External import**: The agent can ingest existing repos/documents into the Externalized Knowledge on demand.
3. **Automatic maintenance**: The agent maintains the knowledge base — not just creates, but updates, prunes, and refines it over time.
4. **Progressive disclosure**: When reading, the agent navigates the knowledge structure incrementally, loading only what's relevant to the current task.

## Progressive Disclosure Algorithm

1. **Query extraction** — Extract keywords/concepts from the current task.
2. **Index lookup** — Query `keywords_index` for candidate entry slugs.
3. **Summary scan** — Read only frontmatter (title, tags, related, first paragraph) of top candidates.
4. **Drill down** — Read full body of promising entries; follow `related` links up to a depth limit (3 hops).
5. **Synthesis** — Compose knowledge from entries with current task context.

## Scope

Two levels with project-level overriding user-level:
- **User-level knowledge**: Lives in `~/.config/agents/knowledge/`. Applies across all projects.
- **Project-level knowledge**: Lives in `.kimi/knowledge/` within the project root. Project-specific knowledge overrides user-level entries with the same slug.

## Maintenance Operations

1. **Staleness detection**: Entries with old `last_updated` + low `access_count` are flagged for archival or deletion.
2. **Deduplication**: Before creating a new entry, the agent checks the index for near-duplicates and merges new information into existing entries.
3. **Cross-reference maintenance**: When an entry is updated, the agent validates `related` links (symmetrical backlinks, no broken refs) and suggests new cross-references for entries sharing keywords but lacking links.
4. **Refinement**: Periodically rewrite entries for clarity and conciseness, preserving meaning.
5. **Confidence scoring (optional)**: Entries carry a confidence score based on source quality and corroboration across sessions. May inform promotion/demotion/archival decisions.

## External Import

The agent can ingest knowledge from multiple surfaces: git repos, local directories, URLs, and raw pasted text. Extraction uses **semantic chunking** — the agent reads the entire source, identifies domain concepts (classes, modules, patterns, decisions, terminology), and creates Knowledge Entries for each. The agent always shows a post-import summary so the user can prune immediately.

## Auto-Extraction Triggers

The agent extracts knowledge when it encounters: resolved domain terms, discovered non-obvious behavior, repeated corrections, settled decisions, or recognized reusable patterns. Extraction is **silent by default** — entries are created without interrupting the user. At session end, the agent shows a summary of what was learned. **Significant entries** (decisions, major patterns) get a brief inline announcement.

## User Interaction

**Always-on with explicit overrides.** Auto-extraction and progressive disclosure run ambiently. Slash commands provide escape-hatch control:
- `/learn <source>` — explicit import from a repo, directory, URL, or raw text
- `/knowledge [query]` — browse or search existing entries
- `/forget <slug>` — delete an entry
- Natural language also works: "learn from this repo," "what do you know about X?"

## File Layout

```
~/.config/agents/knowledge/          ← user-level
├── index.json                        ← Knowledge Index
└── topics/
    └── {domain}/
        └── {slug}.md                ← Knowledge Entry

.kimi/knowledge/                      ← project-level (overrides user-level)
├── index.json
└── topics/
    └── ...
```

The skill itself is a user-level skill:
```
~/.config/agents/skills/externalized-learning/
├── SKILL.md
└── ...
```

## Edge Cases

- **Contradictions**: When a user statement contradicts an existing entry, the agent references the existing knowledge and asks the user to confirm the correction before updating.
- **Index scale**: As the knowledge base grows, progressive disclosure ranks candidates by `access_count` + recency, scanning only the top N. Periodic maintenance archives stale entries and merges thin ones.
- **Circular cross-references**: Expected and allowed. The depth limit (3 hops) prevents infinite loops. No cycle detection needed.
