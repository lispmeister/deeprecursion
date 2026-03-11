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

## Why Not the BEAM?

We spent three posts arguing for Elixir. The argument was sound: the BEAM gives you hot code swapping, supervision trees, process isolation, and automatic rollback. These are real properties that TypeScript doesn't have.

But the [autoresearch pattern](2026-03-11-self-improving-agents.md) exposed a gap. The BEAM catches *crashes*. A supervisor can restart a process that segfaults or throws an exception. What it cannot do is catch a modification that runs without error but produces subtly wrong results — a routing change that slightly degrades response quality, a memory strategy that silently drops important context. These pass the supervisor's test (no crash) while degrading the agent.

The autoresearch pattern requires a judge that's *unambiguous*. The BEAM's judge is "did it crash?" That's necessary but not sufficient. We need a judge that can answer "does this modification satisfy the specification?" — not after running, but *before*.

The Elixir work wasn't wasted. It taught us which runtime properties matter: isolation, rollback, hot loading. These remain requirements. But the judge must be stronger than "it didn't crash." That's what led us to Lean.

---

## Why Lean

Lean's type checker is a different kind of judge. It's not a process you can kill or a file you can overwrite. It's the rules of the language itself. A modification that violates the specification doesn't fail at runtime or get caught in review — it doesn't compile. The separation between judge and subject is enforced by mathematics, not architecture.

When an agent proposes a change to its own code, three questions arise:

1. **Is it safe?** Does it preserve the invariants the system depends on?
2. **Is it correct?** Does it satisfy the specification?
3. **Is it better?** Does it improve the agent's utility?

Lean answers the first two at compile time. The third requires empirical measurement. But having the first two *proven* means the improvement loop only searches over modifications that are already guaranteed safe and correct. The search space shrinks dramatically.

**The judge is the type checker.** Following Part 5's lesson that a singular, unambiguous judge is what makes the autoresearch loop converge: the judge is Lean's type system. Binary — it compiles or it doesn't. Self-observation (log analysis, token efficiency, error rates) and user feedback (weekly reviews) are *signals that inform what to try next*. They help the agent decide which modifications to propose. But the judge that decides whether a modification is *accepted* is the type checker. One judge, multiple signal sources.

---

## The Bootstrap

Lean 4 was bootstrapped this way: first written in C++, then rewritten in Lean, then compiled by itself. We propose a similar arc for Loom — but with an important caveat.

Lean's self-hosting bootstrap was a clean rewrite. The Lean team controlled the entire specification and implementation. Our bootstrap has a fundamentally different shape: we're growing a Lean core inside OpenClaw, a system we don't control, whose interfaces change between releases, and whose agent loop is 57K lines of TypeScript with undocumented edge cases. This isn't a clean rewrite. It's an incremental organ transplant into a moving patient. The risks are categorically different.

With that honest framing:

**Phase 0: Gestation.** The agent lives inside OpenClaw. TypeScript is the host. The Lean core starts as a sidecar process — a single component connected to OpenClaw's gateway via WebSocket. The agent's existing tools, skills, and channels continue to work. The Lean component handles one thing well.

**Phase 1: Organ growth.** More components move to Lean. Each new component connects to the gateway, registers its capabilities, and replaces a TypeScript equivalent. The agent gains verified guarantees one subsystem at a time. OpenClaw shrinks; the Lean core grows.

**Phase 2: Skeleton.** The core agentic loop — decide, act, observe, improve — is rewritten in Lean. The TypeScript gateway becomes a thin bridge to channel adapters. Most reasoning happens in verified code.

**Phase 3: Self-hosting.** The Lean system compiles and runs independently. OpenClaw's channel adapters connect to it as clients, just as they connected to the TypeScript gateway. The host is shed. Loom is born.

Each phase transition depends on the previous one actually working. If the memory organ fails, Phases 2–3 don't happen. Earn the next phase by shipping the current one.

---

## Memory as the Beachhead

Which organ first? Memory. It's OpenClaw's main bottleneck.

The LLMs have no memory of their own. Context is a sliding window. When it fills up, OpenClaw triggers a "memory flush" — a panicked silent turn where the LLM dumps facts to markdown files before they're lost. Retrieval is semantic search over chunks in SQLite. There are no consistency guarantees. Nothing prevents contradictions. Nothing tracks staleness.

A Lean memory service fixes this with types. Four kinds of memory, four sets of invariants:

**User preferences.** "Always use metric units." Stable, accumulate slowly, newer overrides older. Invariant: *preferences are totally ordered by timestamp; a query always returns the most recent value for a given key.*

**Episodic memory.** "Last Tuesday we discussed the tax filing." Timestamped, factual, append-only. Invariant: *episodic memories are immutable once recorded; they can be superseded by corrections but never silently modified.*

**Learned procedures.** "When this user asks about finances, check these three accounts." Behavioral, refined over time. Invariant: *a procedure has a version history; the current version is always reachable; rollback to any previous version is possible.*

**World state.** "The server runs Ubuntu 22.04." Factual, can become stale. Invariant: *every world-state fact has a timestamp and a staleness bound; queries return both the fact and its freshness; expired facts are flagged, not silently served.*

In Lean, these invariants aren't comments or runtime assertions. They're part of the type signature. A function that stores a world-state fact without a staleness bound won't compile. A function that modifies an episodic memory won't compile. The guarantees are structural.

The interface: the Lean memory service connects to OpenClaw's gateway as a WebSocket Node, registers three tools (`memory_store`, `memory_query`, `memory_reflect`), and replaces the current markdown-and-SQLite approach. The agent commits facts to verified storage continuously — so when context compaction happens, nothing important is lost. The panic flush becomes unnecessary.

---

## What's Hard About This

The gap between "specify the memory types" and "build the sidecar" is where this proposal succeeds or fails. Lean 4 is a wonderful language. It is also a language with practical constraints we shouldn't paper over:

**Small ecosystem.** Lean's community is oriented toward mathematics and formal verification, not systems programming. There are no production WebSocket libraries. No battle-tested JSON parsers for gateway protocol frames. Building the sidecar means writing or binding this infrastructure from scratch.

**FFI overhead.** Lean compiles to C, and C FFI exists, but it's not the smooth interop story you get with Elixir's NIFs or Python's ctypes. Connecting to OpenClaw's WebSocket gateway, handling streaming, and maintaining persistent state is non-trivial Lean engineering.

**Compile times.** The autoresearch loop needs fast iteration — modify, compile, evaluate, keep/revert. If Lean's type checker takes minutes to verify a modification, the improvement loop slows to a crawl. Compile time is a hard constraint on how fast the agent can self-improve.

**The moving target.** OpenClaw's gateway protocol isn't versioned or formally specified. It changes between releases. The sidecar must track a moving API surface, and each OpenClaw update might break the bridge. This is the practical cost of growing inside a host you don't control.

None of these are fatal. But they're the actual engineering challenges, and ignoring them would make this proposal dishonest.

---

## What Kills This

Every post in this series has been forward-looking. Here's where it might fall apart:

**The memory invariants are too rigid.** Real conversations are messy. A user says "I prefer metric units" on Monday and "use imperial for this recipe" on Wednesday. Is that a contradiction? A context-dependent override? A preference change? If the type system forces a binary that doesn't match how humans actually manage preferences, the verified memory will be *correct but useless* — technically consistent, practically annoying.

**LLM-generated Lean is syntactically valid but semantically meaningless.** The improvement loop depends on LLMs proposing modifications in Lean. A modification can type-check — satisfy all the invariants — while doing nothing useful. Imagine a memory query function that always returns empty results. It type-checks. It preserves every invariant. It's worthless. The type system catches safety violations; it doesn't catch vacuous implementations. The empirical evaluation must catch these, and that brings back all the noise and ambiguity of measuring "usefulness."

**The sidecar never reaches parity.** OpenClaw's markdown-and-SQLite memory is crude but functional. Users have built workflows around it. The Lean replacement needs to be not just *provably correct* but *practically better* on day one — faster retrieval, fewer lost facts, better staleness handling. If the first version is correct but slow, or correct but missing features users depend on, it won't survive contact with real usage.

**The bootstrap stalls at Phase 1.** The memory organ works but nothing else moves to Lean. The sidecar remains a sidecar. OpenClaw keeps evolving, the bridge keeps breaking, and the Lean component becomes maintenance overhead rather than a foundation. This is the most likely failure mode: not catastrophic failure, but the quiet death of an integration that costs more to maintain than it saves.

---

## What Comes Next

The next move isn't another post. It's the memory service.

1. **Specify the memory types in Lean.** Define the four memory kinds, their invariants, and the store/query/reflect interface. Small enough to write by hand. This is where we discover whether the invariants are too rigid for real conversation.
2. **Build the sidecar.** A Lean binary that connects to OpenClaw's gateway via WebSocket, registers memory tools, and serves verified memory operations. This is where we discover whether Lean's ecosystem is ready for systems work.
3. **Run it.** Replace OpenClaw's memory with the Lean service on a real instance. Measure: fewer contradictions, better retrieval, no panic flushes. This is where we discover whether verified memory is *practically better*, not just *provably correct*.
4. **Start the improvement loop.** The agent observes its own memory performance, proposes modifications to its memory strategies (not the invariants — those are fixed), and keeps what works. This is where we discover whether the autoresearch pattern works for agent self-improvement.

Each step is a falsifiable experiment. If step 1 reveals that the invariants don't fit real conversation, we revise the types. If step 2 reveals that Lean can't handle the systems work, we reconsider the substrate. If step 3 shows no practical improvement over SQLite, we stop.

One working organ is worth more than this entire series. Ship it, measure it, keep or revert.
