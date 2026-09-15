---
title: "Building a Code Intelligence Service for AI Agents"
date: 2026-09-15
categories: engineering
---

## The problem

AI coding agents are now part of daily engineering. They answer questions, review PRs, and write code. But an agent is only as good as its ability to navigate the codebase, and most codebases are not one repository. Ours is dozens of services spread across multiple orgs.

The tools agents had were grep and file reading. Grep cannot tell you who calls a function. Reading files in a loop burns tokens and context on a question like "what breaks if I change this API?" that has a precise, verifiable answer. Agents were guessing, or worse, hallucinating callers and definitions with full confidence.

So I built Kode: a code intelligence service that indexes our repositories on every push and exposes precise code navigation to AI agents over MCP. It now indexes 1,300+ repositories and serves agents across the org.

## What it does

Kode has two layers:

- **The service** owns the index. Symbols, a call graph, and repo records live in Postgres; full-text search shards live in Zoekt, synced to object storage. An indexer pipeline keeps all of it fresh on every push.
- **The MCP layer** is a thin client that serves navigation tools: symbol lookup, blast radius (find all callers of a function across every indexed repo), dependency graphs, service graphs extracted from Helm values, and text search.

A typical agent flow is two calls. `symbol_lookup` finds where a function is defined, then `blast_radius` lists every caller. What used to be a chain of grep-and-read loops is one round trip with a verifiable answer.

## Architecture

The production flow is event driven:

```
GitHub push
  -> webhook        receives push event, publishes to a queue
  -> Kafka          repo events, ordered per partition
  -> consumer       clone/fetch, parse, index
  -> Postgres       symbols + call graph
  -> Zoekt          text-search shards
```

The consumer clones via go-git using short-lived GitHub App installation tokens. Parsing is SCIP for Go, which gives import-path-accurate call edges, and a tree-sitter fallback for Lua, TypeScript, Python, and Rust. The MCP layer runs as a separate deployment behind an API gateway with key-based auth, with a search sidecar alongside it.

Deployment is Kubernetes: event receiver, consumer, MCP server, and search as separate deployments, schema migrations and backfills as jobs, GitOps for rollouts, and preview environments that spin up a full stack per pull request.

## Design decisions worth explaining

The architecture is standard event-driven plumbing. The design decisions are where the interesting work happened.

### 1. Self-healing event ordering

Webhooks arrive out of order. It is a fact of distributed systems, and the obvious fix is ancestry checking: is the incoming commit a descendant of the last indexed commit? We deliberately do not do this. If an event is stale, indexing it is wasted work but never incorrect state, because the next push reconciles everything. This removes coordination and edge-case handling from the consumer entirely. A rare redundant re-index costs less than the complexity of preventing it.

### 2. Precise call graphs where it matters, fast parsing everywhere else

SCIP (Sourcegraph's index protocol) gives Go an exact call graph: edges resolved by import path, not by name. Tree-sitter for other languages is faster to wire up but name-matches call edges, which is less accurate. Rather than block on perfect parsing for every language, we ship precise edges for the language with the most code and accept approximation elsewhere. The tool surface tells the client which edges are resolved and which are name-matched, so agents know what to trust.

### 3. Security as a property of tokens, not of network position

Clone auth uses GitHub App installation tokens, resolved per org and cached, scoped to the App's repository selection. Nothing long-lived exists in the pipeline. Repos outside the token's scope are skipped, not indexed, which makes the index boundary and the access boundary the same line.

### 4. Incremental indexing that cannot starve

A full backfill touches 1,300+ repos. A naive worker pool lets one enormous repo monopolize workers while hundreds of small repos wait. The indexer uses a worker pool with memory-aware backoff: workers shed load when memory pressure rises, so a heavy push degrades throughput gracefully instead of getting OOM-killed mid-index.

### 5. MCP-native from day one

This is not a REST API with an MCP adapter bolted on. The tool surface was designed for how agents actually query: exact symbol lookups, batched lookups, blast radius as a first-class query, and text search for everything else. Fewer, denser tool calls matter more than a rich API, because every round trip is latency and tokens in an agent loop.

## Why build this instead of buying it

Code intelligence is not a new problem, and there are established products for it. So why did we build our own?

**Commercial code intelligence is priced for enterprises, not for agents.** Products like Sourcegraph offer excellent code search and graph features, but they are sold per seat to human users, with enterprise contracts to match. Our consumer is not a human with a budget line; it is an agent making thousands of lookups a day across the whole org. A per-seat model fits that workload badly, either because agent "seats" are not a category most vendors price for, or because the volume of queries an agent loop generates dwarfs what a human team would ever do.

**The expensive parts turned out not to be that expensive.** The core stack here is Postgres, Zoekt (open source, the same text indexer that powers GitHub code search), SCIP, and tree-sitter, all running on a modest Kubernetes footprint. The genuinely hard work was in the design decisions above, and those transfer to anyone willing to own a small Go service. What vendors sell as a platform, we assembled as a pipeline plus two hundred lines of API.

**MCP changes the calculus.** Existing code search products expose REST APIs and web UIs designed for humans. Agents need a different interface: exact symbol resolution, batched lookups, blast radius as a query, and answers dense enough that no follow-up archaeology is needed. Buying a great code search engine and bolting MCP on top of its human-oriented API gets you a worse agent experience than designing the tool surface for agents first.

**Owning the index is the feature.** When the index is yours, new questions are new queries, not feature requests. Extracting service dependencies from Helm values, labeling name-matched call edges, and tuning indexing for our repo shapes all took days because we own the pipeline. On a vendor platform, each of those is a support ticket, an enterprise plan tier, or an impossibility.

None of this means commercial tools are wrong in general. If your need is a human-facing code search UI and you want zero ownership, buy one. But for agent-facing code intelligence at org scale, a self-hosted index with an MCP-native surface is dramatically cheaper and, more importantly, yours to evolve.

## What we learned

**Trustworthiness beats coverage.** An agent acts on what you tell it. A wrong blast radius is worse than no blast radius, because the agent will confidently plan around it. This is why we label name-matched edges and why precision was prioritized over indexing every language perfectly.

**Design for reconciliation, not prevention.** The no-ancestry-check decision is the clearest example. Most of the complexity we considered adding was there to prevent states that were harmless and self-correcting. Deleting that complexity made the system simpler and more reliable at the same time.

**Preview environments pay for themselves.** PR-preview stacks for an indexer-heavy service sound expensive. They are not, and they changed how changes get reviewed: a reviewer can index a real repo set against a real candidate build before merge.

## What is next

- Cheaper incremental re-indexing with prune-generation tracking
- SCIP precision extended beyond Go
- Deeper service-graph extraction from configuration, so agents can answer "what talks to what in production" alongside "what calls what in code"

## Wrap-up

Code intelligence is becoming infrastructure. Agents do not need more context windows; they need better questions answered faster. Point an agent at an index like this and "who calls this function" stops being a research task and becomes a lookup.
