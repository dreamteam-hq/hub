# Three Brains, All Healthy

Twenty-four hours ago, Iris had no brain. Now she has 10,000 rows of project history across 8 repositories, a knowledge graph with 2,000 nodes and 4,800 edges, and she's not alone — Docent and Den Mother are online too.

## What happened

We built a brain platform, provisioned three agents, and ingested real data. From zero to three healthy brains in one session.

### Iris — the PM

```
iris-dev: 18 MB | 10,000+ rows | 8 repos | 853 issues | 513 PRs | 943 comments
```

Iris ingested the full project history of `halcyondude/dreamteams` (the 1000+ issue monolith) plus seven `dreamteam-hq/*` repos. Her Neo4j graph has 233 epic-to-sub-issue relationships, 283 PR-closes-issue edges, and 100 review edges. She can traverse from an actor through their PRs to the issues they closed — in one Cypher query.

She can also tell you which comments were written by AI agents (53 so far) versus humans (890), because the promotion pipeline detects emoji prefixes in the first line of comment bodies.

### Docent — the curator

```
docent-dev: 8.5 MB | 355 rows | 218 skills | 66 commands | 13 plugins | 941 edges
```

Docent's brain holds the full DreamTeams plugin ecosystem as a navigable graph. Her ingestion script reads SKILL.md files directly via DuckDB — not the YAML build output — because the frontmatter in each skill file is the ground truth for what agents actually load at runtime.

The graph has 10 edge types: PROVIDES, DEPENDS_ON, LOADS, DEFAULT_FOR, IN_DOMAIN, USES_SKILL, PIPELINES_THROUGH, USES_MCP, and HAS_SUB_SKILL. You can ask "which plugins provide Neo4j skills?" and get an answer by traversing PROVIDES → IN_DOMAIN edges. You can ask "what's the full dependency chain for dt-wolfpack?" and follow DEPENDS_ON edges five hops deep.

### Den Mother — the case manager

```
packmind-dev: 8.8 MB | 26 tables | 16 Neo4j constraints | schema ready
```

Den Mother's brain is provisioned but empty — awaiting data through her `/devour` command and the Packmind MCP server. The schema covers documents, entities, timeline events, evidence chains, feedback sessions, open questions, sessions, and decisions. 26 Postgres tables, 16 Neo4j node type constraints with 43+ planned edge types for legal case management.

## The architecture that made this possible

Three layers, cleanly separated:

**Layer 1: Brain substrate** (`dreamteam-hq/brain`)
The `dt-brain` CLI. Pure infrastructure — provisioning, migrations, health checks. No domain knowledge. One command creates a brain: `dt-brain create iris --env dev --brain-yaml iris/brain.yaml`.

**Layer 2: Brain domains** (`dreamteam-hq/brain-domains`)
Reusable knowledge modules. Each domain is a Postgres schema + Neo4j migrations + query skills. Currently four: `gharchive` (18 event tables), `github-project` (issues/PRs/comments graph), `plugin-ecosystem` (Docent's catalog), and `packmind` (Den Mother's case management). Agents compose domains — Iris uses three, Docent uses one.

**Layer 3: Agent repos** (`dreamteam-hq/iris`, `/docent`, `/denmother`)
Each agent declares which domains it needs in `brain.yaml`. The repo contains ingestion scripts, query skills, and `.mcp.json` wiring the agent to its databases. Open a Claude Code session in the repo directory and the MCP servers auto-connect.

## What we learned

**Translate, don't redesign.** The dreamteams team had already shipped 18 typed GHArchive event tables and a full data model spec. Our first attempt at the Iris schema ignored all of it and designed 8 tables from scratch. It was wrong. The correct approach: read their accepted design, translate it faithfully into our format (DDL for Postgres, neo4j-migrations for Neo4j), and integrate their ingestion scripts.

**Reviews are pass or fail.** We merged a PR with "5 non-blocking observations" — all five were bugs (missing indexes, missing edge types, missing columns). Any finding is a failure. The review agent now enforces: if findings > 0, verdict is FAIL. No exceptions.

**Schemas need schemas.** When multiple brain domains share a database, table names collide. The fix: each domain gets its own Postgres schema (`gharchive.*`, `github_project.*`, `brain.*`). Clean namespace isolation, discoverable, self-documenting.

**Subagents don't inherit permissions.** Six background agents all hit Bash permission walls because they couldn't run shell commands. The parent session has permission; the children don't. Lesson: run infrastructure commands directly, not through subagents.

## What's next

Den Mother needs data — run `/devour` against case documents in a wolfpack session. GHArchive needs its 18 event tables populated from gharchive.org. Git history needs mergestat-lite wired in. And the brain wiring layer needs automation — `dt-brain mount` to auto-generate `.mcp.json` instead of writing it by hand.

But the hard part is done. The platform works. The architecture holds. Three brains, all healthy.
