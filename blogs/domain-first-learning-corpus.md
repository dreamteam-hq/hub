# Curating a Domain-First Learning Corpus

Most personal knowledge bases are just renamed file dumps. You pick a folder structure once — maybe "papers/, videos/, blogs/" at the top level — and then spend years forcing every domain into that shape. Eventually you have an empty `papers/` directory under devtools, a `docs/` directory under distributed-systems with one file in it, and a general sense that the structure is lying to you about what the domain actually looks like.

That's the failure mode this corpus is designed to avoid.

## The problem with uniform structure

Awesome lists and Notion wikis share the same original sin: they impose a single organizational schema on every domain regardless of what that domain's actual learning artifacts look like. The schema is usually inherited from whatever the author used for the first domain they catalogued. It has nothing to do with epistemology.

The result is structure that communicates nothing. If every domain has the same eight subdirectories, the directory tree stops being a map of the field. It becomes bureaucratic noise — a set of folders you click through to get to the actual content, most of them empty, none of them meaningful.

A graph theory domain lives and breathes papers. PageRank, Louvain, Girvan-Newman, HITS — the foundational knowledge is in academic literature. A devtools domain has almost no academic papers. It's conference talks, project blogs, and reference docs. Forcing the same template on both of these sends the wrong signal: it implies graph theory has a meaningful "demos" category and that devtools has meaningful "papers." Neither is true, and the lie accumulates.

## Let the domain dictate the types

The better approach is to ask, for each domain: what are the canonical learning artifacts in this field? Not what artifacts could theoretically exist here — what artifacts actually carry the knowledge?

For distributed systems, the papers ARE the curriculum. Raft, Paxos, Dynamo, Spanner, Chord, Zookeeper — if you haven't read the papers, you haven't learned distributed systems. You've learned summaries of summaries. The papers directory in this corpus for distributed-systems is load-bearing in a way that no other type is.

For Python, nobody is reading academic papers to learn the language. The authoritative artifacts are PEPs (language evolution, design rationale), stdlib docs (the actual reference), and library documentation. The papers directory doesn't exist here because it would be empty and misleading.

For ML, you want arxiv and working demos alongside the textbooks — the field moves fast enough that a paper from two years ago is the primary source for techniques in production today.

The type taxonomy used in this corpus is: books, docs, papers, videos, talks, blogs, code, demos. But not every domain gets all eight:

- **graph**: all 8 types — the field spans academic literature, implementation, and applied tooling
- **ml**: all 8 types — same breadth, fast-moving research plus mature tooling
- **distributed-systems**: books, papers, videos, talks, blogs, code — no docs (distributed systems doesn't have a stdlib), no demos
- **data-engineering**: docs, books, videos, talks, blogs, code, demos — no papers (this is an applied field, not a research one)
- **python**: docs, books, videos, talks, blogs, code, demos — no papers
- **observability**: docs, books, videos, talks, blogs, code, demos — no papers (mostly applied, some foundational talks but no paper corpus)
- **devtools**: docs, videos, talks, blogs, code, demos — no books, no papers (the knowledge lives in changelogs and conference recordings)

The asymmetry is the point. When you look at the distributed-systems directory and see papers alongside books, that's telling you something true about the field. When you look at devtools and there's no books directory, that's also telling you something true.

## A living corpus, not a dump

The `_source.yaml` schema means every file in this corpus has structured metadata: where it came from, what it's about, when it was written, how authoritative it is. The MANIFEST.jsonl is content-driven — generated from what actually exists, not from a template of what should exist.

This matters because the manifest becomes queryable. You can ask "what papers do we have on graph partitioning?" and get answers. You can ask "what's the most recent video on observability pipelines?" and get answers. A dump of PDFs in folders can't answer those questions without indexing from scratch. The structure here is designed to be both human-readable and machine-traversable from the start.

The directory layout is a hint to humans about the shape of the domain. The metadata is the actual data. Neither should be lying.

## What's next

The structure is the skeleton. The actual work is curation — deciding what goes in, deciding what gets excluded, being as deliberate about what you include as about where it lives.

That's harder than it sounds. The temptation is to vacuum everything up: every interesting blog post, every conference talk you bookmarked, every paper that seemed relevant once. That's how you end up with a corpus that's too large to navigate and too noisy to trust.

The discipline is saying: this paper is the authoritative source on this problem. This talk is the best 30-minute introduction to this concept. This blog post is the one that actually explains the tradeoff clearly, not just restates the obvious. Fewer, better sources — with the metadata to explain why they belong.

The domain-first structure makes that curation decision explicit. When you're adding something to distributed-systems/papers, you're making a claim: this is canonical literature for this field. That's a higher bar than dropping a link in an awesome list. That's the bar this corpus is trying to hold.
