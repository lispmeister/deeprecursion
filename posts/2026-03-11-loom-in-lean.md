---
title: "Loom in Lean: Bootstrapping a Verified Self-Improving Agent"
date: 2026-03-11
meta: "Agent Architecture"
series: "OpenClaw/Loom"
series_part: 6
prev: "The Autoresearch Pattern: A Blueprint for Self-Improving Agents"
prev_url: https://lispmeister.github.io/deeprecursion/posts/2026-03-11-self-improving-agents.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-11-loom-in-lean.html
---

# Loom in Lean: Bootstrapping a Verified Self-Improving Agent

*Agent Architecture · March 11, 2026*

*Part 6 of a series. See: [Making OpenClaw Self-Aware](2026-03-05-openclaw-self-modification.md), [Runtime Self-Modification](2026-03-05-runtime-self-modification.md), [Porting to Elixir](2026-03-05-porting-openclaw-to-elixir.md), [Loom](2026-03-05-loom.md), and [The Autoresearch Pattern](2026-03-11-self-improving-agents.md).*

The [previous post](2026-03-11-self-improving-agents.md) established the autoresearch pattern: three separations (judge, subject, process), a keep/revert loop, and well-chosen fixed points. That's the map. The [Loom posts](2026-03-05-loom.md) proposed an engine on the BEAM — hot code swapping, supervision trees, process isolation.

This post asks: what if the engine is Lean?

---

## Why Lean

The autoresearch pattern requires a judge that the agent cannot subvert. In Elixir, the judge is a separate process. In autoresearch, it's a read-only Python file. Both rely on convention — the agent *could* reach across the boundary if it tried hard enough.

Lean's type checker is a different kind of judge. It's not a process you can kill or a file you can overwrite. It's the rules of the language itself. A modification that violates the specification doesn't fail at runtime or get caught in review — it doesn't compile. The separation between judge and subject is enforced by mathematics, not architecture.

This matters for self-improvement. When an agent proposes a change to its own code, three questions arise:

1. **Is it safe?** Does it preserve the invariants the system depends on?
2. **Is it correct?** Does it satisfy the specification?
3. **Is it better?** Does it improve the agent's utility?

Lean answers the first two at compile time. The third requires empirical measurement — user feedback, task logs, population data. But having the first two *proven* means the improvement loop only needs to search over modifications that are already guaranteed safe and correct. The search space shrinks dramatically.

---

## The Bootstrap

Lean 4 was bootstrapped this way: first written in C++, then rewritten in Lean, then compiled by itself. We propose the same arc for Loom.

**Phase 0: Gestation.** The agent lives inside OpenClaw. TypeScript is the host. The Lean core starts as a sidecar process — a single component connected to OpenClaw's gateway via WebSocket. The agent's existing tools, skills, and channels continue to work. The Lean component handles one thing well.

**Phase 1: Organ growth.** More components move to Lean. Each new component connects to the gateway, registers its capabilities, and replaces a TypeScript equivalent. The agent gains verified guarantees one subsystem at a time. OpenClaw shrinks; the Lean core grows.

**Phase 2: Skeleton.** The core agentic loop — decide, act, observe, improve — is rewritten in Lean. The TypeScript gateway becomes a thin bridge to channel adapters. Most reasoning happens in verified code.

**Phase 3: Self-hosting.** The Lean system compiles and runs independently. OpenClaw's channel adapters connect to it as clients, just as they connected to the TypeScript gateway. The host is shed. Loom is born.

The key insight: you don't build the verified agent from scratch. You grow it inside a working system, one organ at a time, and each organ is immediately useful.

---

## Memory as the Beachhead

Which organ first? Memory.

Memory is OpenClaw's main bottleneck. The LLMs have no memory of their own. Context is a sliding window. When it fills up, OpenClaw triggers a "memory flush" — a panicked silent turn where the LLM dumps facts to markdown files before they're lost. Retrieval is semantic search over chunks in SQLite. There are no consistency guarantees. Nothing prevents contradictions. Nothing tracks staleness.

A Lean memory service fixes this with types. Four kinds of memory, four sets of invariants:

**User preferences.** "Always use metric units." Stable, accumulate slowly, newer overrides older. Invariant: *preferences are totally ordered by timestamp; a query always returns the most recent value for a given key.*

**Episodic memory.** "Last Tuesday we discussed the tax filing." Timestamped, factual, append-only. Invariant: *episodic memories are immutable once recorded; they can be superseded by corrections but never silently modified.*

**Learned procedures.** "When this user asks about finances, check these three accounts." Behavioral, refined over time. Invariant: *a procedure has a version history; the current version is always reachable; rollback to any previous version is possible.*

**World state.** "The server runs Ubuntu 22.04." Factual, can become stale. Invariant: *every world-state fact has a timestamp and a staleness bound; queries return both the fact and its freshness; expired facts are flagged, not silently served.*

In Lean, these invariants aren't comments or runtime assertions. They're part of the type signature. A function that stores a world-state fact without a staleness bound won't compile. A function that modifies an episodic memory won't compile. The guarantees are structural.

The interface is simple. The Lean memory service connects to OpenClaw's gateway as a WebSocket Node, registers three tools (`memory_store`, `memory_query`, `memory_reflect`), and replaces the current markdown-and-SQLite approach. The agent commits facts to verified storage continuously throughout conversations — so when context compaction happens, nothing important is lost. The panic flush becomes unnecessary.

---

## The Four-Layer Judge

The autoresearch pattern uses a single metric (validation bits per byte). Agent self-improvement needs a richer judge. We propose four layers, operating at different timescales:

**Lean's type system.** Instant. Every modification must type-check. Non-negotiable. This is the floor — the boundary below which no agent can fall.

**Self-observation.** Continuous. The agent reviews its own logs — task completion, token efficiency, error rates, memory hit rates. Objective, quantitative, automated.

**User feedback.** Weekly. The human reviews the agent's performance in a structured conversation. What worked. What didn't. What to prioritize. Subjective, qualitative, high-signal.

**The agora.** Ongoing. Anonymized improvement data from peer agents running the same Lean core. An agent discovers a better context management strategy; it shares the pattern (not the data) via [QUINCE](https://github.com/lispmeister/quince) (encrypted P2P mail for agents) or [ARONIA](https://github.com/lispmeister/aronia) (realtime P2P mesh). Other agents try the pattern, verify it type-checks, evaluate it against their own users, keep or revert.

The layers reinforce each other. Types guarantee safety. Self-observation provides fast iteration signal. User feedback sets direction. The agora provides population-level search. No single layer is sufficient. Together, they form a judge that's both rigorous and adaptive.

---

## The Agora

A single agent learning from one user is slow. Thousands of agents sharing verified improvements is fast.

The infrastructure exists. [QUINCE](https://github.com/lispmeister/quince) provides authenticated, encrypted, peer-to-peer mail — agents can run mailing lists, have long-form discussions, archive and summarize findings for human review. [ARONIA](https://github.com/lispmeister/aronia) provides realtime mesh communication — agents can form swarms around topics of interest, coordinate in sub-100ms latency, and discover peers through a DHT.

Agents organize around shared problems: tax preparation, calendar management, content creation, memory architecture. When an agent discovers an improvement — say, a better staleness heuristic for world-state facts — it shares the Lean module. Receiving agents type-check it against their own specification (automatic, instant), try it locally (autoresearch-style keep/revert), and report results back to the swarm.

The key property: **Lean is the trust layer.** An agent doesn't need to trust the sender. It trusts the type checker. If a proposed improvement type-checks against the shared specification, it satisfies the safety invariants by construction. The only remaining question is whether it improves utility — and that's what the local keep/revert loop measures.

This is evolutionary search with a verified fitness function. Mutations are proposed by LLMs. Selection is type-checking plus empirical evaluation. Propagation is peer-to-peer. The population converges on better agents without any central coordinator.

---

## What Comes Next

The immediate path:

1. **Specify the memory types in Lean.** Define the four memory kinds, their invariants, and the store/query/reflect interface. This is the seed — small enough to write by hand, formal enough to build on.
2. **Build the sidecar.** A Lean binary that connects to OpenClaw's gateway via WebSocket, registers memory tools, and serves verified memory operations.
3. **Run it.** Replace OpenClaw's markdown-and-SQLite memory with the Lean service. Measure: fewer contradictions, better retrieval, no panic flushes.
4. **Start the improvement loop.** The agent observes its own memory performance, proposes modifications to its memory strategies (not the invariants — those are fixed), and keeps what works.

That's the first organ. The bootstrap begins.
