---
name: externalized-learning
description: Build and maintain externalized knowledge for the agent across sessions. Agent auto-extracts domain concepts, patterns, decisions, and gotchas during conversation; imports from repos/docs/URLs on demand; maintains the knowledge base (dedup, staleness, cross-refs); reads via progressive disclosure. Use when agent needs to remember, recall, or build long-term knowledge from conversations and external sources. Triggered by /learn, /knowledge, /forget, or natural language like "learn from this," "what do you know about X," "remember this."
---

# Externalized Learning

Build, maintain, and query the agent's externalized knowledge base. See [CONTEXT.md](CONTEXT.md) for the full glossary and domain model.

## Quick start

```
User: "learn from this repo"
Agent: ingests via semantic chunking → creates entries → shows import summary

User: "what do you know about OIDC?"
Agent: index lookup → summary scan → drill-down → synthesizes answer
```

## User commands

- `/learn <source>` — import from repo, directory, URL, or raw text
- `/knowledge [query]` — browse or search entries
- `/forget <slug>` — delete an entry
- Natural language equivalents also work

## Workflows

### 1. Progressive disclosure (reading)

When the current task could benefit from existing knowledge:

1. Extract keywords/concepts from the task
2. Query `index.json` → `keywords_index` for candidate slugs
3. Read only frontmatter (title, tags, related, first paragraph) of top candidates
4. Drill into full body of promising entries; follow `related` links up to 3 hops
5. Synthesize with current task context

Rank candidates by `access_count` + recency. Update `last_accessed` and `access_count` on read.

### 2. Auto-extraction (writing)

During conversation, silently extract knowledge when encountering:

- Resolved domain terms → glossary entry
- Non-obvious behavior / gotchas → gotcha entry
- Repeated corrections → correction entry
- Settled architectural decisions → decision entry
- Recognized reusable patterns → pattern entry

Before creating: check index for near-duplicates. If found, merge into existing entry.
Announce inline only for significant entries (decisions, major patterns).
At session end, summarize what was learned.

### 3. External import

Ingest from git repos, local directories, URLs, or raw text.
Use semantic chunking: read the full source, identify domain concepts, create an entry per concept.
Always show a post-import summary so the user can prune.

### 4. Maintenance (automatic)

- **Deduplication**: before creation, check for near-duplicates; merge if found
- **Cross-reference maintenance**: validate `related` links after updates; suggest new links for entries sharing keywords but lacking refs
- **Staleness**: flag entries with old `last_updated` + low `access_count` for archival/deletion
- **Refinement**: periodically rewrite entries for clarity, preserving meaning

### 5. Handling contradictions

When user statement contradicts an existing entry: reference the existing knowledge inline and ask the user to confirm before updating.

## Entry format

```markdown
# Canonical Title

**Tags**: #glossary #pattern
**Last updated**: 2026-05-22
**Related**: [[other-slug]], [[another-slug]]

(free-form body)
```

## Index format

`index.json` with four sections: `entries`, `tags` (inverted), `keywords_index` (inverted), `usage` (per-entry stats).

## File locations

- User-level knowledge: `~/.config/agents/knowledge/`
- Project-level knowledge: `.kimi/knowledge/` (overrides user-level)
- Each has `index.json` + `topics/{domain}/{slug}.md`
