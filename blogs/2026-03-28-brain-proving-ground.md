# From Corpus to Brain: Building the Proving Ground

You're running a law practice, and your filing system is a disaster. Client files are in one cabinet, case law is in another, and the calendar is on the wall. You need three things to prepare for a hearing: the client's sworn statements, the relevant statutes, and the docket. Every time you pull a case together, you're walking between three cabinets, cross-referencing by hand, and hoping you didn't miss something.

Now imagine five lawyers sharing one office, and none of them organized their cabinets the same way.

That's the problem the brain platform solves — except the lawyers are AI agents, and the cabinets are databases.

## Why four stores, not one

The most common question about the brain architecture is: why not just use one database?

The short answer is that different kinds of information need different kinds of storage, the same way your office needs both filing cabinets (for documents) and a whiteboard (for mapping relationships between cases). Cramming everything into one system means the system is mediocre at everything and excellent at nothing.

Here's how it breaks down:

```
┌─────────────────────────────────────────────────────────────┐
│                      An Agent's Brain                       │
│                                                             │
│  ┌─────────────────────────┐  ┌───────────────────────────┐ │
│  │       SCRATCH PAD       │  │     PERMANENT RECORD      │ │
│  │   (session — cleared    │  │   (persistent — survives  │ │
│  │    when you're done)    │  │    between sessions)      │ │
│  │                         │  │                           │ │
│  │  ┌───────────────────┐  │  │  ┌─────────────────────┐  │ │
│  │  │      DuckDB       │  │  │  │     PostgreSQL      │  │ │
│  │  │                   │  │  │  │                     │  │ │
│  │  │  Staging area.    │  │  │  │  The filing cabinet │  │ │
│  │  │  Import, clean,   │  │  │  │  Documents, records │  │ │
│  │  │  compare across   │  │  │  │  structured data    │  │ │
│  │  │  sources.         │  │  │  │  with strict rules. │  │ │
│  │  └───────────────────┘  │  │  └─────────────────────┘  │ │
│  │                         │  │                           │ │
│  │  ┌───────────────────┐  │  │  ┌─────────────────────┐  │ │
│  │  │    LadybugDB      │  │  │  │      Neo4j          │  │ │
│  │  │                   │  │  │  │                     │  │ │
│  │  │  Draft board.     │  │  │  │  The relationship   │  │ │
│  │  │  Sketch out       │  │  │  │  map. Who connects  │  │ │
│  │  │  connections,     │  │  │  │  to whom, what      │  │ │
│  │  │  test theories.   │  │  │  │  depends on what.   │  │ │
│  │  └───────────────────┘  │  │  └─────────────────────┘  │ │
│  └─────────────────────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**The filing cabinet (PostgreSQL)** stores structured records — documents, metadata, entity tables. Think of it as the system of record. When you need to answer "show me every motion filed in March," this is where it lives. It has strict rules about what goes where (schemas), and those rules are version-controlled so you can track every change.

**The relationship map (Neo4j)** stores connections. A filing cabinet can tell you that Motion A and Statute B both exist, but it's terrible at answering "what are all the things connected to this custody standard, and what's connected to *those*?" Graph databases are built for exactly this — following chains of relationships across dozens of hops without breaking a sweat.

**The staging area (DuckDB)** is where raw material lands before it's ready for the permanent record. Import a CSV, a PDF extraction, a YouTube transcript — it all goes here first to be cleaned, validated, and shaped. When you're done, the good data promotes to the filing cabinet or the relationship map. Then the staging area resets. No cleanup, no leftover drafts.

**The draft board (LadybugDB)** is the graph equivalent of scratch paper. Want to test a theory about how concepts connect before committing it to the permanent knowledge graph? Sketch it here. If it works, promote it. If it doesn't, close the session and it's gone.

The left column is disposable. The right column is permanent. The top row handles structured records. The bottom row handles relationships. Four quadrants, zero overlap.

## How information flows through a brain

Every piece of information follows the same path, and the first two steps never involve AI:

```
    Raw Input (book, transcript, API data, legal filing)
         │
         ▼
    ┌─────────┐
    │ EXTRACT  │  Pull out the structured parts.
    │          │  No AI — just parsing.
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │  SHAPE   │  Clean it, normalize it, validate it.
    │          │  No AI — just rules.
    └────┬────┘
         │
         ├──────────────────────┐
         ▼                      ▼
    ┌─────────┐           ┌──────────┐
    │CLASSIFY │           │ PROMOTE  │  Move clean data into
    │  (AI)   │           │          │  permanent storage.
    └────┬────┘           └──────────┘
         │
         ▼
    ┌─────────┐
    │ REASON  │  AI does the thinking —
    │  (AI)   │  connections, patterns,
    └────┬────┘  implications.
         │
         ▼
    ┌─────────┐
    │ PERSIST │  Store the AI's conclusions
    │         │  in permanent storage.
    └─────────┘
```

This is the single most important design decision in the whole system: **Extract and Shape are deterministic.** No AI. No judgment calls. Given the same input, you get the same output, every time. You can test it. You can reproduce it. You can trust it.

AI enters at Classify (quick categorization) and Reason (deep analysis). But by the time AI touches the data, it's already been extracted and validated. The AI adds intelligence on top of structure — it doesn't create the structure. This is the difference between building on a foundation and building on sand.

## Every brain has a name

Every database instance — across every agent, every environment, every engine — follows one naming rule:

```
{who}-{where}-{what}

iris-dev-postgres       Iris's development filing cabinet
iris-dev-neo4j          Iris's development relationship map
wolfpack-prod-postgres  Wolfpack's production filing cabinet
learning-dev-neo4j      Learning corpus's development graph
```

This sounds trivial. It isn't. When you're managing ten brains across five agents, the ability to type `iris-dev-*` and find everything that belongs to Iris's development environment — or `*-prod-neo4j` and find every production graph — is the difference between organized and overwhelmed.

## One command to rule them all

Setting up a brain used to mean manually creating databases, running migration scripts, writing config files, and hoping you didn't miss a step. Now it's one command:

```
$ dt-brain create iris --env dev

[1/5] Creating filing cabinet (iris-dev-postgres) ...... done
[2/5] Applying filing rules ............................ done  (3 migrations)
[3/5] Creating relationship map (iris-dev-neo4j) ....... done
[4/5] Applying graph constraints ....................... done  (2 migrations)
[5/5] Writing config ................................... done

Brain "iris-dev" is ready.
```

If step 3 fails, steps 1 and 2 get rolled back automatically. You never end up with half a brain.

The CLI handles the daily workflow too:

- **`dt-brain ls`** — see all your brains, their sizes, whether their schemas are current
- **`dt-brain health`** — spot problems before they compound (runaway growth, slow queries, stuck processes)
- **`dt-brain snapshot`** — save a complete copy of both databases before making changes
- **`dt-brain clone`** — duplicate a brain for testing without touching the original

## The agents who use these brains

Each agent in the DreamTeams ecosystem gets its own brain, sized and structured for its job:

| Brain | Agent | What it does |
|-------|-------|-------------|
| `iris` | Iris | Project management — issues, sprints, standups |
| `docent` | Docent | Documentation catalog — plugin ecosystem, guides |
| `denmother` | Den Mother | Legal case management — the Wolfpack |
| `learning` | (corpus) | Knowledge graph — everything we're learning |
| `nerdherd` | NerdHerd | Algorithms, ML, graph neural networks |

Den Mother's brain knows about custody statutes, filing deadlines, and case relationships. Iris's brain knows about sprint velocity and issue triage. They share the same architecture — four stores, same naming convention, same migration tools, same health checks — but each brain's *content* is completely different.

This is the point of the platform: write the infrastructure once, then every new agent gets a brain that works the same way. No reinventing the wheel. No "how does this agent store its data?" questions. The answer is always the same four stores, the same CLI, the same rules.

## What the proving ground proved

The learning repo was the test bed. Before any of this shipped to the brain platform, it had to work against real data:

- **600+ corpus items** across seven domains — graph theory, ML, observability, data engineering, distributed systems, developer tools, Python
- **Ingestion pipelines** for books, YouTube transcripts, GitHub history, and MA legal statutes — each following the deterministic Extract → Shape → Classify → Reason → Persist pipeline
- **Cross-source queries** — joining YouTube transcript concepts with book chapter topics with GitHub commit patterns, all through the DuckDB staging layer
- **Real graph traversals** — prerequisite chains, concept maps, cross-source relationship detection in Neo4j

19 concepts, 14 code samples, 6 cross-source relationships emerged from just two test manifests. The patterns work. The architecture holds.

## What's next

The brain repo has its architecture spec written and its CLI taking shape. The next milestones:

1. **Ship the CLI** — `dt-brain create` is working; the remaining verbs (`migrate`, `health`, `snapshot`) are in progress
2. **Provision the first real brain** — Iris gets a brain, backed by the platform instead of hand-wired connections
3. **Go multi-brain** — cross-brain federation through DuckDB, so agents can query each other's knowledge when it makes sense
4. **Workshop review** — the architecture spec gets its design review, then it's the blueprint going forward

The total cost of running all of this: **$18.90 per month.** One database IDE license and one schema management tool. Everything else — PostgreSQL, Neo4j, DuckDB, LadybugDB, the migration tools, the monitoring — is free.

The proving ground proved the patterns. Now they ship to every agent in the fleet.
