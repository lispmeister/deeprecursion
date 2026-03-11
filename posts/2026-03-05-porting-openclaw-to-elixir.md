---
title: "Porting OpenClaw's Core to Elixir"
date: 2026-03-05
meta: "Agent Architecture"
series: "OpenClaw/Loom"
series_part: 3
prev: "Runtime Self-Modification: Beyond TypeScript"
prev_url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-runtime-self-modification.md
next: "Loom: The Case for Building on Opal"
next_url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-loom.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-porting-openclaw-to-elixir.html
---

# Porting OpenClaw's Core to Elixir

*Agent Architecture · March 5, 2026*

*Part 3 of a series. See: [Making OpenClaw Self-Aware](2026-03-05-openclaw-self-modification.md) and [Runtime Self-Modification: Beyond TypeScript](2026-03-05-runtime-self-modification.md).*

The [previous post](2026-03-05-runtime-self-modification.md) argued that building a self-modifying agent in TypeScript is a fundamental mismatch — the language treats runtime code modification as an error condition. Elixir/BEAM treats it as a design requirement. So what would it actually take to port [OpenClaw](https://docs.openclaw.ai) to Elixir?

We looked at the codebase. Here's what we found.

## What We're Looking At

OpenClaw is ~470K lines of TypeScript across 2,712 source files. It's a monorepo containing a core gateway, 40+ channel extensions (Telegram, Slack, Discord, WhatsApp, Signal, Matrix, and more), 54+ skills, native apps, and a web UI. The architecture is a WebSocket/HTTP hub that routes messages between messaging platforms and an LLM agent loop, with a plugin system for dynamic skill and tool registration. State is file-based JSON plus SQLite-vec for vector embeddings.

## The Strategy: Port the Core, Keep the Rest

Porting all 40+ channel adapters to Elixir would mean rewriting SDKs that only exist in the Node ecosystem. That's pointless work. The right move: port only the core to Elixir, keep the existing TypeScript channels, skills, and UI, and bridge them.

**What moves to Elixir:**

- **Gateway hub** — message routing, connection management → Phoenix Channels + OTP supervisors
- **Agent loop** — LLM invocation, tool execution, approval workflows, context management → GenServer per session
- **Plugin/skill runtime** — discovery, loading, hook lifecycle, tool registration → behaviours + dynamic module loading
- **State layer** — config, sessions, auth profiles → ETS/GenServer with file persistence

**What stays in TypeScript:**

- All 40+ channel adapters
- All 54+ skills
- Native apps and web UI

**The bridge:** The channel plugins already speak WebSocket to the gateway. The Elixir core exposes the same local WebSocket endpoint, existing TypeScript plugins connect as clients. Minimal changes on the TS side.

## What You Get

The whole point is that OTP gives you for free what OpenClaw hand-builds in TypeScript:

- **Hot code swapping** — modify the agent loop at runtime, no restart, no dropped connections
- **Process isolation** — a buggy skill or compromised agent session can't take down the gateway
- **Supervision trees** — automatic recovery from failures, with rollback to previous module versions
- **Distribution** — multi-node agent clusters via `Node.connect/1`, no infrastructure changes
- **Two-version coexistence** — old and new code run simultaneously during upgrades

The self-modification problem from the first post becomes tractable: the agent proposes a module change, the supervisor hot-loads it, and if it crashes, rolls back automatically. No git branches, no rebuild, no restart.

> **The silent failure case:** hot code swapping catches crashes. It doesn't catch a modification that runs without error but produces subtly wrong results — a context strategy that silently drops important facts, a routing change that slightly degrades response quality. For those, you need a stronger guarantee than "it didn't crash."

---

## Effort Estimate: Humans Only

| Component | Estimate |
|---|---|
| Core gateway + routing | 3–4 weeks |
| Agent loop + tool execution | 4–6 weeks |
| Plugin/skill runtime | 2–3 weeks |
| State management | 1–2 weeks |
| Node bridge layer | 2–3 weeks |
| Integration + testing | 2–3 weeks |
| **Total** | **3–4 person-months** |

## Effort Estimate: With Coding Agents

Assume access to the latest coding agents (Opus 4.6, GPT-5.3 Codex) with unlimited token budget. Agents compress the *typing* time dramatically but don't eliminate the *thinking* time.

**What agents handle well:** translating well-understood TypeScript patterns to idiomatic Elixir, writing the bridge protocol, state management boilerplate, test generation and porting.

**What still needs a human architect:**

- **Supervision tree design** — which processes supervise which, restart strategies, how agent sessions map to process lifecycles. 2–3 days of thinking.
- **Bridge contract** — where the boundary sits between Elixir and TypeScript, what crosses the wire. 1–2 days.
- **Hot code swapping semantics** — which modules are hot-swappable, how `code_change/3` handles state migration. This is the whole reason for the port. 2–3 days of design.
- **Integration debugging** — when the Elixir core and Node sidecar disagree about message format or sequencing.

| Component | Humans only | With agents |
|---|---|---|
| Core gateway + routing | 3–4 weeks | 4–5 days |
| Agent loop + tool execution | 4–6 weeks | 1–2 weeks |
| Plugin/skill runtime | 2–3 weeks | 3–4 days |
| State management | 1–2 weeks | 2–3 days |
| Node bridge layer | 2–3 weeks | 3–5 days |
| Integration + testing | 2–3 weeks | 1–2 weeks |
| **Total** | **3–4 months** | **4–6 weeks** |

The agent loop is the component that doesn't compress as much — it has the most subtle behavior (context window management, tool approval chains, LLM response streaming) and the most room for bugs that only surface at integration time.

> **The real bottleneck becomes review.** With unlimited tokens, agents can produce the entire Elixir codebase in a week. But someone needs to verify the supervision tree is right, the failure modes are handled, and the hot-swap paths actually work. That review and iteration is ~60% of the remaining calendar time. One human architect plus agents: ~5–6 weeks wall clock.

*Continue reading: [Loom: The Case for Building on Opal](2026-03-05-loom.md)*
