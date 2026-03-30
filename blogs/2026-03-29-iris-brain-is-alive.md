# Iris Has a Brain

Iris's brain is alive. Not a metaphor — a real, queryable, four-store database system that holds her project history as both structured records and a navigable graph.

Here's what it took, and how you'd do it yourself.

## What's running

Two databases, one brain:

```
$ dt-brain ls

BRAIN   TYPE   ENV   PG     NEO4J   PG SIZE   PG ROWS   NEO4J NODES   NEO4J EDGES
iris    iris   dev   ✓ up   ✓ up    100 kB    344       87            165
```

**Postgres** holds the structured records — 344 rows across 36 tables in three schemas:
- `brain.*` — shared infrastructure (query ledger, watermark, module registry)
- `gharchive.*` — 18 typed event tables for GH Archive data (empty until we run that ingestion)
- `github_project.*` — issues, pull requests, comments, timelines, and relationship tables

**Neo4j** holds the knowledge graph — 87 nodes and 165 edges representing who authored what, which PRs closed which issues, and who commented where.

## The graph

This is what Iris sees when she looks at `dreamteam-hq/learning`:

![Iris brain graph — Actor at center with AUTHORED edges to Issues and PRs](iris-brain-graph-1.png)

One actor (halcyondude) at the center, radiating out to 22 issues (purple), 28 pull requests (green), and 35 comments (teal). The edges tell the story: 50 AUTHORED relationships, 35 COMMENTED, 25 ON (comment → parent), 3 CLOSES (PR → Issue), 2 REVIEWED.

Some queries that work right now:

**Which PRs closed issues?**
```cypher
MATCH (pr:PullRequest)-[:CLOSES]->(i:Issue)
RETURN pr.number AS pr, pr.title, i.number AS issue, i.title
```

Returns three rows — PR #47 closed the brain proving ground epic (#43), PR #44 closed the agentskills restructure handoff (#42), PR #26 closed the content staging spike (#22).

**Full traversal — who authored PRs that closed issues?**
```cypher
MATCH (a:Actor)-[:AUTHORED]->(pr:PullRequest)-[:CLOSES]->(i:Issue)
RETURN a.login, pr.number, pr.title, i.number, i.title
```

**Most commented issues:**
```cypher
MATCH (c:Comment)-[:ON]->(i:Issue)
RETURN i.number, i.title, count(c) AS comments
ORDER BY comments DESC LIMIT 5
```

The Workshop Primer (#49) and the MVVM for Godot research (#24) top the list with 3 comments each.

## How to reproduce this

### Prerequisites

```bash
brew install postgresql@18
brew install cypher-shell
brew install ariga/tap/atlas
brew install michael-simons/homebrew-neo4j-migrations/neo4j-migrations
```

Neo4j Desktop installed with a local DBMS running (password: `dreamteam`). APOC and GDS plugins installed via the Desktop UI.

### Clone the repos

```bash
cd ~/gh/dreamteam-hq
git clone git@github.com:dreamteam-hq/brain.git
git clone git@github.com:dreamteam-hq/brain-domains.git
git clone git@github.com:dreamteam-hq/iris.git
```

### Provision the brain

```bash
cd brain
uv run dt-brain create iris --env dev \
  --brain-yaml ~/gh/dreamteam-hq/iris/brain.yaml
```

This resolves 4 domains (brain, gharchive, github-project, git-history), creates the Postgres database with all schemas, creates the Neo4j database, and applies constraints.

### Ingest data

```bash
cd ~/gh/dreamteam-hq/iris/scripts

# Step 1: GitHub GraphQL → DuckDB
GITHUB_TOKEN="$GITHUB_PERSONAL_ACCESS_TOKEN" \
  uv run --with duckdb python3 ingest_github_history.py \
  --repo dreamteam-hq/learning \
  --db /tmp/iris-ingest/github.duckdb

# Step 2: DuckDB → Postgres
uv run --with duckdb --with psycopg2-binary python3 -c "
import duckdb
conn = duckdb.connect('/tmp/iris-ingest/github.duckdb')
conn.execute('INSTALL postgres_scanner; LOAD postgres_scanner;')
conn.execute(\"ATTACH 'dbname=iris-dev-postgres' AS pg (TYPE postgres);\")
for t in ['issues','pull_requests','comments','pr_closes','pr_reviews','timeline_events']:
    count = conn.execute(f'SELECT count(*) FROM {t}').fetchone()[0]
    if count > 0:
        conn.execute(f'INSERT INTO pg.github_project.{t} SELECT * FROM {t}')
        print(f'  {t}: {count} rows')
conn.close()
"

# Step 3: Postgres → Neo4j graph
uv run --with neo4j --with psycopg2-binary python3 promote_to_neo4j.py
```

### Verify

```bash
dt-brain ls                    # shows iris-dev with row/node/edge counts
dt-brain health                # both stores healthy
psql -d iris-dev-postgres -c "SELECT schemaname, count(*) FROM pg_tables
  WHERE schemaname IN ('brain','gharchive','github_project')
  GROUP BY schemaname;"        # 3 schemas, 36 tables
```

Open Neo4j Browser at `http://localhost:7474`, connect to `iris-dev-neo4j`, and run:
```cypher
CALL apoc.meta.graph()
```

You'll see the schema — Actor, Issue, PullRequest, Comment, Repository — and all the edges between them.

## Update: Three brains, all healthy

Since the original post, two more brains came online:

```
BRAIN        TYPE      PG ROWS    NEO4J NODES    NEO4J EDGES
iris-dev     iris      9,800+     2,000+         4,600+
docent-dev   docent    355        407            941
packmind-dev packmind  0          3              2 (schema ready)
```

**Iris ingested `halcyondude/dreamteams`** — the full project history. 816 issues, 499 PRs, 905 comments, 233 sub-issue relationships, 6,768 timeline events. Plus the learning repo data. She can now answer questions about the entire project.

**Docent's brain is live** with the full plugin ecosystem graph — 218 skills, 66 commands, 13 plugins, 17 domains, 941 edges. Her ingestion script reads SKILL.md files directly via DuckDB as ground truth, building the graph with UNWIND + MERGE. She can answer "which plugins provide Neo4j skills?" or "show me the dependency chain for dt-wolfpack" by traversing her graph.

**Den Mother's brain is provisioned** — 26 Postgres tables in the `packmind` schema, Neo4j constraints for 16 node types. Schema ready, awaiting data. Her ingestion runs through the `/devour` command and the Packmind MCP server.

## The architecture

Three layers, cleanly separated:

```
dt-brain (substrate)                    dreamteam-hq/brain
    consumed by
brain domains (data models + skills)    dreamteam-hq/brain-domains
    consumed by
agents (pick which domains they need)   dreamteam-hq/iris, /docent, /denmother
```

Each agent declares which brain domains it needs in `brain.yaml`. `dt-brain create` resolves those domains, composes all schemas, and provisions everything. Brain domains are reusable — Iris and Sia both consume `gharchive` and `github-project`.

All agents are wired to their databases via `.mcp.json` — Postgres MCP + Neo4j MCP, auto-connecting when you open a Claude Code session in the agent's repo.

## What's next

- **Den Mother data ingestion** — run `/devour` against case documents to populate the packmind brain
- **GHArchive events** — 18 typed event tables waiting for data from gharchive.org
- **Git history** — mergestat-lite integration for commit-level analysis
- **Brain wiring automation** — `dt-brain mount` to auto-generate `.mcp.json` from the registry
- **Cross-brain federation** — DuckDB attaching multiple Postgres databases for cross-agent queries

Three brains, all healthy. The proving ground proved the patterns. Now they ship.
