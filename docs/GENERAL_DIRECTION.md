# Ora — General Direction

2026-11-13 · Markdown copy of the shared doc: https://claude.ai/artifact/HbnZzEAGMqf47fEErsX3A1

## Principle

**Ora is one semantic graph of the codebase, with two faces: humans see and govern it, agents query and commit to it.**

Ora replaces both the code editor and Git. Its unit is not a file or a line but a graph node: a data shape, a function, an edge, an effect. Every change is an operation on nodes that carries its intent and its evidence.

The constraint Ora attacks is human attention. Compute now scales code production and store throughput; it does not scale a human's ability to understand what changed. Ora makes change legible, prevents conflicts before they happen, and asks the human to look only where an invariant is at stake.

## Why: one bottleneck, two broken tools

Git and the editor both assume a human author typing at human pace. Agents broke both assumptions at once, and the faster store systems built in 2026 fix only half of it.

| Area | Assumption that broke | What happens under agents |
| --- | --- | --- |
| Version control | The unit of change is a line of text | Lockfiles and adjacent lines conflict falsely; a changed signature plus a new caller merges cleanly and breaks |
| Version control | Conflicts are found at merge | The agent's work is already spent; nothing shows in-flight changes |
| Version control | One working copy per person | Worktrees isolate files only; ports, databases and caches still collide; about 5 to 7 parallel agents is a reported ceiling |
| Version control | Writes are rare | Servers serialise unrelated writes; Anthropic's Scion went from about 2,500 to over 30,000 merges a day |
| Version control | A commit is a snapshot and a sentence | Prompt, assumptions and test evidence are lost; history is either noise or a lossy squash |
| Version control | Verification follows pushes | CI repeats the same work instead of testing what changed |
| Version control | Permissions stop at repo or path | An agent's scope is unbounded |
| Editor | The human holds a cursor in a buffer | The human no longer types |
| Editor | The file tree organises understanding | Execution paths cut across it |
| Editor | Review is a line diff sorted by filename | PR size up 154%, review time up 91%; one study finds 61% of agent PRs get no review |
| Editor | One author at a time | No view of what ten agents are doing now, or where |
| Editor | The reader wrote the code | Humans lose the mental model of code they never wrote |

Figures come from secondary write-ups; see Sources.

**Why viewer and VCS are one system.** Scion and OpenAI's Pando made the store faster, which only moves the pile to the reviewer. The real limit is that the unit of change is illegible, and that is a property of the store's data model. What the human views is change, intent and evidence, which only exist if the store records them. What the human does in the viewer is a store operation: a lasso is a write lease, a boundary is a law, an approval is a seal. Agents need a viewer too: today they grep files and rebuild structure every session; a graph query is their view.

## What Ora is not

- **Not a text editor.** There is no cursor-and-buffer editing surface; humans read, select, declare and approve.
- **Not a coding agent.** Ora hosts any agent through one interface and never generates code itself.
- **Not a compiler or test runner.** It projects files for the existing toolchain and records the results.
- **Not Git-compatible in its write path.** A Git adapter imports existing repos and exports sealed commits, at the edge only.
- **Not built on embeddings.** The graph is deterministic; a local embedding index only maps natural language to exact nodes.
- **Not a hosted platform first.** It runs locally for one developer and many agents before it runs as a shared server.

## System diagram

One graph sits under both faces; every write passes a lease and the gate. Arrows are calls; each layer depends only on the layers below it.

```
   ┌──────────────────────────────┐            ┌──────────────────────────────┐
   │ Humans                       │            │ Coding agents                │
   │ own Layer 0 · review only    │            │ any agent, many in parallel  │
   │ exceptions                   │            │                              │
   └──────────────┬───────────────┘            └──────────────┬───────────────┘
                  │ see, select, approve                      │ query, claim, commit
                  ▼                                           ▼
   ┌──────────────────────────────┐            ┌──────────────────────────────┐
   │ viewer · web canvas          │            │ agent API · MCP tools        │
   │ call slices · state ribbon · │            │ query slices · claim leases  │
   │ semantic diff · scrubber ·   │            │ commit ops · request the gate│
   │ lasso · live lease map       │            │                              │
   └──────────────┬───────────────┘            └──────────────┬───────────────┘
                  ▲▼                                          ▲▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │ server · one API for both faces, plus a live event stream                │
   └─────────────────────────────────────┬────────────────────────────────────┘
                                         │ routes every request
                                         ▼
   ┌─ core services ──────────────────────────────────────────────────────────┐
   │ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐  │
   │ │ query         │ │ lease         │ │ workspace     │ │ gate          │  │
   │ │ slices, blast │ │ claims on     │ │ per-agent view│ │ boundaries,   │  │
   │ │ radius, diff  │ │ subgraphs,    │ │ edits become  │ │ tests, seal   │  │
   │ │               │ │ refuses       │ │ ops           │ │ or escalate   │  │
   │ │               │ │ overlaps      │ │               │ │               │  │
   │ └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘  │
   └─────────────────────────────────────┬────────────────────────────────────┘
                                         │ reads and writes the graph
                                         ▼
   ┌─ graph engine ───────────────────────────────────────────────────────────┐
   │ ┌──────────────────────┐ ┌──────────────────────┐ ┌─────────────────────┐│
   │ │ frontend             │ │ identity             │ │ store               ││
   │ │ tree-sitter:         │ │ matches facts to     │ │ append-only op log  ││
   │ │ structure, fast      │ │ stable ids; survives │ │ content-addressed   ││
   │ │ SCIP: resolution     │ │ rename and move      │ │ nodes               ││
   │ │ at the gate          │ │                      │ │                     ││
   │ └──────────────────────┘ └──────────────────────┘ └─────────────────────┘│
   └─────────────────────────────────────┬────────────────────────────────────┘
                                         ▲▼ files in, sealed commits out
   ┌─ edge adapters · outside the core ───────────────────────────────────────┐
   │ ┌──────────────────────┐ ┌──────────────────────┐ ┌─────────────────────┐│
   │ │ source files ·       │ │ gitbridge            │ │ embed               ││
   │ │ toolchain            │ │ imports an existing  │ │ maps prompts to     ││
   │ │ projected out by     │ │ Git repo; exports    │ │ node ids; lookup    ││
   │ │ workspace, parsed    │ │ sealed commits       │ │ only, never truth   ││
   │ │ back in by frontend  │ │                      │ │                     ││
   │ └──────────────────────┘ └──────────────────────┘ └─────────────────────┘│
   └──────────────────────────────────────────────────────────────────────────┘
   ╔══════════════════════════════════════════════════════════════════════════╗
   ║ graph · the one model every module shares                                ║
   ║ Layer 0 contract and state · Layer 1 execution and effect ·              ║
   ║ Layer 2 temporal and collaboration                                       ║
   ╚══════════════════════════════════════════════════════════════════════════╝
```

Humans and agents reach the same server through their own face. Every change flows down through a lease, a workspace and the gate into the store; files exist only at the edge, for the toolchain.

## Core model

One graph, three layers, and one rule for identity. Every other module reads or writes only this model.

| Layer | Node and edge kinds | Who authors it |
| --- | --- | --- |
| 0 · Contract and state | `State` (struct, class, interface, schema, field), `Boundary` (subsystem and what it may read or mutate) | `State` is extracted; `Boundary` is declared by humans in a boundary file |
| 1 · Execution and effect | `Transform` (function, method), `Edge` (calls, reads, writes, returns, constructs), `Effect` (pure read, mutates field, allocates, I/O) | Extracted by the compiler |
| 2 · Temporal and collaboration | `Op` (micro-commit), `Seal` (architectural commit), `Devlog` (intent, assumptions, receipts), `Lease` (live claim on a subgraph) | Written by agents and the gate |

**Node identity.** A node keeps its id through edits, renames and moves. The id is minted at creation and carried by every later op; the compiler matches extracted nodes to existing ids by qualified symbol name first, then by structural similarity. A split or merge is an explicit op that records the lineage. Diff, devlog, leases and history all depend on this rule, so it is designed first.

**Provenance.** Every edge and effect carries how it was established: `resolved` (from a semantic index), `heuristic` (from syntax alone) or `unknown`. The gate treats anything not `resolved` as the worst case, for example "may mutate".

**Files are a projection.** The graph is canonical. Each workspace materialises files on demand for compilers, tests and editors, and file edits are parsed back into ops.

The schema is language-agnostic from the start; per-language knowledge lives only in front-end adapters.

## Modules and interfaces

Ten modules in four layers; each layer depends only on the ones below it. Interfaces are named here; their exact signatures come with each stage's design.

| Module | Purpose | Surface | Depends on |
| --- | --- | --- | --- |
| `graph` | The schema: node, edge, effect and op types | Types plus pure functions; no I/O | nothing |
| `store` | Content-addressed nodes and an append-only op log | `Apply(op)`, `Snapshot(rev)`, `Log(nodeIDs)` | `graph` |
| `frontend` | Source to graph: `tree-sitter` for structure, SCIP for resolution | `Extract(files) → facts`; one adapter per language | `graph` |
| `identity` | Matches extracted facts to existing node ids | `Reconcile(facts, snapshot) → ops` | `graph` |
| `query` | Slices, blast radius, effect search, semantic diff | `Slice`, `BlastRadius`, `Diff(revA, revB)` | `store` |
| `workspace` | A per-agent view: projects files out, parses edits back to ops | `Open(lease)`, `Materialise`, `Ingest(edits)` | `store`, `frontend`, `identity` |
| `lease` | Live claims on subgraphs; conflicts refused at claim time | `Claim(nodes, intent)`, `Release`, `Watch` | `store`, `query` |
| `gate` | Boundary checks, affected-test runs, seal or escalate | `Check(ops) → receipt`, `Seal(ops)` | `query`, `lease`, `store` |
| `server` | One API for both faces: agent protocol and viewer stream | Agent API (also exposed as MCP tools), viewer event stream | all above |
| `viewer` | Web canvas: call slices, ribbon, diff, scrubber, lasso, map of live leases | Talks only to `server` | `server` |

Two edge adapters sit outside the core: `gitbridge` (import a Git repo, export a sealed commit) and `embed` (local natural-language lookup to node ids).

## Key flows

**An agent change, from intent to seal.** There are no branches and no merges: one live graph, with leases preventing conflicts up front.

1. A human lassos nodes in the viewer and attaches a prompt, or an agent requests a slice by query.
2. `lease` claims the slice. A claim that overlaps a live lease, or a contract that lease depends on, is refused, and the refusal names the holder.
3. `workspace` opens a view of the slice and projects files out for the agent's tools.
4. Each agent edit becomes a micro-commit op, with a devlog entry for intent and assumptions.
5. The agent requests the gate. `frontend` re-indexes with SCIP so the facts become `resolved`, then `gate` checks boundaries, effect escalations and lease scope, and runs only the tests the graph links to the changed nodes.
6. A pass with no contract change seals automatically: the micro-commits squash into one architectural commit with its receipts. Anything else goes to the human queue.
7. The lease is released, and other agents' workspaces see the new revision.

**Human review by exception.** Humans author Layer 0 and approve only what touches it. The queue shows four things and nothing else: changed contracts, effect escalations (a pure reader that now mutates or does I/O), boundary crossings, and regions with unresolved facts. Each item opens as a semantic diff with the devlog beside it and the micro-commits on a scrubber.

**Analysis at two speeds.** On every micro-commit, `tree-sitter` updates structure in milliseconds and tolerates broken code. At the gate, SCIP gives full resolution; code must compile to seal anyway, so this costs nothing extra.

## Decisions

| Decision | Chosen | Why | Rejected |
| --- | --- | --- | --- |
| Version control | Ora's own store; Git only as an edge adapter | Git's line unit, merge-time conflicts and lost provenance are the bottleneck; even Anthropic and OpenAI built their own | Git underneath with shadow refs: it inherits the line model and merge-time conflicts |
| Viewer and VCS | One system over one graph | The viewer's data (intent, evidence, change) exists only if the store records it; viewer actions are store operations | Separate products joined by an API: the store would stay illegible |
| Implementation language | Go | A concurrent server at heart; explicit code that humans can review; graphs are easy without ownership rules | Rust: arenas and index juggling for graphs; slower to review |
| Language scope | Agnostic from the start, validated on two unlike languages | A schema shaped by one language would have to be rebuilt | One language first: faster, but bakes in its assumptions |
| Syntax layer | `tree-sitter` | Grammars for nearly every language, tolerates broken code, exact byte ranges, incremental | ANTLR (slower, weaker recovery); per-language compilers (not agnostic); LLVM IR (compiled languages only) |
| Semantic layer | SCIP indexes | Batch, deterministic, stable global symbol names, no long-running processes | LSP (one query at a time, one heavy server per workspace); kept as a fallback adapter |
| Change model | Leases on subgraphs, no branches | Conflicts are prevented at claim time instead of found at merge | Branches with semantic merge: still discovers conflicts after the work is done |
| Review model | Review by exception | Human attention is the scarce resource | Review every change: does not scale to agent volume |
| Viewer stack | Thin TypeScript canvas served by the Go binary | All graph logic stays in one place, on the server | Native GUI: slower to build, no shared view across a team |

**Prior art to study:** Joern's code property graph (a close match to Layers 0 and 1); Scion, Pando, Kin, AVCS and Lit for agent-native version control.

## Stages

Each stage ships both faces and proves one thing with a measurement. Stage 0 is this document plus the schema, and needs approval before code.

| Stage | Agent face | Human face | Proves (the measurement) |
| --- | --- | --- | --- |
| 0 | Graph schema, identity rule, store model written up | This document | The design is approved |
| 1 | `ora build` plus graph queries for context | Call-slice canvas, state ribbon | Extraction accuracy on two real repos in two languages, against a hand-checked sample |
| 2 | Store with micro-commits and devlog | Semantic diff and time-scrubber | Identity survives rename, move and split; diff is smaller than `git diff` for the same change |
| 3 | Leases and workspaces | Lasso intent; live map of in-flight work | N local agents run with zero merge conflicts; throughput measured against Git worktrees |
| 4 | Gate, receipts, auto-seal | Boundary file; exception queue | Share of changes sealed without a human, and the escaped-defect rate |
| 5 | Remote canonical graph | Multi-developer presence | Leases and seals work across machines |

## Risks and open questions

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Effect tagging is imprecise, worst in dynamic languages | Review by exception only saves attention if "pure reader" can be trusted | Provenance on every fact; unresolved facts escalate; per-language precision adapters later |
| Node identity breaks on large refactors | Diff, devlog and leases all hang on stable ids | Explicit split and merge ops; measure identity survival in Stage 2 |
| SCIP has no call edges | Layer 1 needs them | Derive calls from references inside call expressions found by `tree-sitter` |
| SCIP indexers vary in upkeep and in read/write marking | Coverage claims depend on them | Test each indexer on real repos before relying on it |
| `tree-sitter` needs cgo in Go | Complicates cross-compiling | Subprocess or WebAssembly grammars; maturity of the WebAssembly route is unverified |
| Adoption: teams must leave Git | Scion's authors say review, search and permissions all had to migrate | `gitbridge` export keeps existing CI and hosting working |
| Lease granularity | Too coarse serialises agents; too fine misses contract-level conflicts | Lease a slice plus a read dependency on the contracts it uses |

**Open questions**

- [ ] Which two languages validate Stage 1?
- [ ] Which real repos are the benchmark?
- [ ] Boundary file format: its own syntax or an existing one?
- [ ] Agent protocol: MCP only, or a native API as well?

## Sources

- [Why AI coding agents change source control: worktrees, Pando and Scion](https://betterstack.com/community/guides/ai/agent-version-control/)
- [Why merge conflicts became the new agentic bottleneck](https://adamtornhill.substack.com/p/why-merge-conflicts-became-the-new)
- [The fourth generation of version control](https://cherry.vc/news/the-fourth-generation-of-version-control)
- [The end is nigh for code review (Semgrep)](https://semgrep.dev/blog/2026/end-is-nigh-for-code-review)
- [Why agent-generated code is breaking the pull request review model](https://www.softwareseni.com/why-agent-generated-code-is-breaking-the-pull-request-review-model/)
- [Git worktrees changed how I run parallel AI agents](https://news.mewayz.com/news/167933/Technology/March-24-2026/Git-Worktrees-Changed-How-I-Run-Parallel-AI-Agents)
- [Is the IDE dead? (Addy Osmani)](https://addyosmani.com/blog/death-of-the-ide/)
