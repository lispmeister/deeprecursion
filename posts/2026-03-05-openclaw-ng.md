---
title: "Loom: The Case for Building on Opal"
date: 2026-03-05
meta: "Agent Architecture"
series: "OpenClaw/Loom"
series_part: 4
prev: "Porting OpenClaw's Core to Elixir"
prev_url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-porting-openclaw-to-elixir.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-openclaw-ng.html
---

# Loom: The Case for Building on Opal

*Agent Architecture · March 5, 2026*

*Part 4 of a series. See: [Making OpenClaw Self-Aware](2026-03-05-openclaw-self-modification.md), [Runtime Self-Modification: Beyond TypeScript](2026-03-05-runtime-self-modification.md), and [Porting OpenClaw's Core to Elixir](2026-03-05-porting-openclaw-to-elixir.md).*

This series started with a question: [how can an AI agent patch its own code?](2026-03-05-openclaw-self-modification.md) We worked through [why TypeScript is the wrong substrate](2026-03-05-runtime-self-modification.md) for runtime self-modification, then [estimated the effort](2026-03-05-porting-openclaw-to-elixir.md) to port OpenClaw's core to Elixir. The conclusion was 4–6 weeks with coding agents.

Then we found [Opal](https://github.com/matteing/opal).

## What Opal Already Is

Opal is an Elixir port of the Pi coding agent — the same agent that sits at the core of OpenClaw. It's ~4,500 lines of clean OTP code with 87 test files. The architecture is not a sketch; it's a working system.

The agent loop is a `:gen_statem` with four explicit states: **idle → running → streaming → executing_tools**. Each session gets its own supervision subtree: a `Task.Supervisor` for concurrent tool execution, a `DynamicSupervisor` for sub-agents, an optional MCP supervisor, and the agent process itself. Tools implement an `Opal.Tool` behaviour. Providers implement an `Opal.Provider` behaviour. Sessions persist to DETS with branching support.

Here's what's already implemented:

| Component | Status | Notes |
|---|---|---|
| Agent state machine | Complete | `:gen_statem`, 651 lines |
| Tool framework | Complete | 10 built-in tools (read, write, edit, grep, shell, sub-agent, tasks, ask-user, skills, debug) |
| LLM streaming | Complete | Provider-agnostic SSE parsing |
| Concurrent tool execution | Complete | `Task.Supervisor` with crash isolation |
| Sub-agent spawning | Complete | Depth-limited child agents |
| Session persistence | Complete | ETS + DETS with conversation branching |
| Context compaction | Complete | LLM-driven summarization at token threshold |
| Retry & backoff | Complete | Classifies retryable vs. permanent errors |
| MCP integration | Complete | Tools + resources via Anubis client |
| JSON-RPC transport | Complete | Bidirectional stdio protocol |
| Supervision tree | Complete | Per-session subtree, `:rest_for_one` |
| Provider abstraction | Partial | Only GitHub Copilot built-in; framework is extensible |

## What This Means for the Port

The hardest architectural decision in any Elixir port of OpenClaw is decomposing the agent loop. OpenClaw's `pi-embedded-runner/run.ts` is a single 57,000-line file containing the entire agent lifecycle: history management, tool execution, token counting, context window management, LLM invocation with retry and failover, session compaction. Porting that monolith to OTP processes — deciding which parts become GenServers, how they supervise each other, where state lives — is the kind of work that resists automation. It requires understanding both the source semantics and the target paradigm deeply enough to make structural decisions that will hold up under production load.

Opal has already made those decisions. And made them well.

The 57K-line `run.ts` becomes a 651-line `:gen_statem` plus ~850 lines of supporting modules (streaming, tool execution, compaction, overflow detection). The session lane serialization that OpenClaw builds explicitly — queuing runs per session to prevent race conditions — falls out naturally from the GenServer mailbox. One process per session, messages queue by default.

Mapping Opal's coverage against the four core areas we identified for porting:

| Core Area | OpenClaw Lines | Opal Coverage | Remaining Work |
|---|---|---|---|
| Agent Loop | ~12K (+ 57K `run.ts`) | ~80% | Approval gates, multi-provider auth failover |
| Plugin/Skill Runtime | ~5K | ~30% | Plugin discovery/registry, hook lifecycle, install/update |
| Gateway Hub | ~23K | ~5% | WebSocket server, HTTP, routing, auth, broadcast |
| State Layer | ~20K | ~20% | Config schemas, secrets, memory/embeddings |

The agent loop is 80% done. That's the component our [previous estimate](2026-03-05-porting-openclaw-to-elixir.md) said would compress the least under coding agents because of its subtle behavior. It's the component with the most room for bugs that only surface at integration time. And it's already built and tested.

---

## The Self-Modification Payoff

This is where the whole series converges.

We [argued](2026-03-05-runtime-self-modification.md) that the BEAM gives you structural safety for runtime self-modification: process isolation contains blast radius, supervision trees provide automatic rollback, hot code swapping lets you modify behavior without dropping connections, and two-version module coexistence makes upgrades atomic. None of this is possible in TypeScript.

Opal makes this concrete. The agent is a `:gen_statem`. You can hot-swap its module while it's running. The `code_change/4` callback lets you migrate state between versions. If the new code crashes, the supervisor restarts the agent with the previous version. The tool executor runs in a separate `Task.Supervisor` — a bad tool can't take down the agent. Sub-agents run in their own `DynamicSupervisor` — a rogue sub-agent can't take down its parent.

The self-modifying agent scenario from our first post becomes tractable:

- Agent identifies a bug in its own tool execution logic
- Agent generates a new version of the `ToolRunner` module
- System hot-loads the new module via `:code.load_binary/3`
- If it crashes, supervisor restarts with the old version — automatically
- If it works, the agent continues with the improved code — no restart, no lost context

In OpenClaw's TypeScript, this same scenario requires: write to file, git branch, `pnpm build`, restart gateway, lose all active sessions. The BEAM makes self-modification a runtime operation. Opal gives us the supervision structure to make it safe.

---

## The Build Plan

Starting from Opal, building Loom incrementally:

**Phase 1: Complete the agent core.** Add Anthropic and OpenAI providers (implement the `Opal.Provider` behaviour, reuse the shared OpenAI helpers already in the codebase). Add approval gates to tool execution. Add auth profile failover. This gives us a fully functional multi-provider coding agent with safety controls.

**Phase 2: Add the gateway.** Phoenix app wrapping Opal's agent into WebSocket channels. Each connected client gets a `SessionServer`. Implement OpenClaw's frame protocol (req/res/event JSON frames) so existing clients can connect. Binding router as pattern matching on message metadata. HTTP endpoints for health, Canvas, and the OpenAI-compatible chat completions API.

**Phase 3: Add the plugin system.** Plugin discovery scanning filesystem paths. Plugin loading via `:code.load_binary/3` (replacing TypeScript's Jiti). Plugin registry in ETS. Hook lifecycle via `:telemetry`. Elixir behaviours define the plugin contract. Skills hot-reload is already in Opal.

**Phase 4: Bridge to TypeScript.** The gateway exposes the same WebSocket endpoint that OpenClaw's TypeScript channel adapters already connect to. The 40+ existing adapters (WhatsApp, Telegram, Slack, Discord, Signal, etc.) connect as clients. Minimal changes on the TypeScript side — they already speak WebSocket with JSON frames.

**Phase 5: Complete the state layer.** Config schema validation with NimbleOptions. Secrets module with encrypted file storage. Memory and vector embeddings via NIF or Erlang port to SQLite-vec.

---

## Effort Estimate

One human architect, frontier coding agents (Opus 4.6, GPT-5.3 Codex), unlimited token budget.

| Phase | Scope | Estimate |
|---|---|---|
| 1. Complete agent core | Providers, approval gates, auth failover | 3–4 days |
| 2. Gateway | Phoenix WebSocket/HTTP, routing, auth, broadcast | 5–7 days |
| 3. Plugin system | Discovery, loading, registry, hooks | 3–4 days |
| 4. TypeScript bridge | Frame protocol compatibility, channel adapter connection | 3–5 days |
| 5. State layer | Config schemas, secrets, memory/embeddings | 3–4 days |
| Integration & testing | End-to-end validation, edge cases, load testing | 5–7 days |
| **Total** | | **~4–5 weeks** |

Compare to our previous estimates:

| Approach | Estimate |
|---|---|
| Port from scratch, humans only | 3–4 months |
| Port from scratch, with coding agents | 4–6 weeks |
| **Build on Opal, with coding agents** | **4–5 weeks** |

The calendar time compression from Opal is modest — about a week — because the gateway and plugin system still need to be built regardless. But the *risk* compression is significant. The agent loop is the component most likely to harbor subtle bugs (race conditions in streaming, edge cases in compaction, tool execution ordering). Opal has 87 test files covering these paths. Starting from tested, working code for the hardest component means fewer integration surprises in weeks 3–5.

> **The real argument for Opal isn't time savings. It's that someone already made the hard architectural decisions correctly.** The `:gen_statem` agent, the per-session supervision tree, the `Task.Supervisor` for tool isolation, the Registry-based pub/sub, the branching session model — these are exactly the choices we would have made. Starting from Opal means we're building on validated design decisions rather than re-deriving them.

---

## Why This Is Worth Doing

Three reasons.

**1. Runtime self-modification becomes a feature, not a hack.** OpenClaw's TypeScript architecture requires a full build-restart cycle to change agent behavior. On the BEAM, the agent can modify its own tools, swap its own modules, and upgrade its own capabilities — all while maintaining active sessions. The supervision tree ensures that a bad modification is automatically rolled back. This is the enabling capability for the self-improving agent we described in the [first post](2026-03-05-openclaw-self-modification.md).

**2. The concurrency model is right for multi-channel agents.** OpenClaw routes messages from 20+ platforms through a single Node.js event loop. The BEAM gives each session its own lightweight process, each channel its own process, each tool execution its own supervised task. Backpressure, isolation, and fault tolerance are structural. A slow LLM response on one session doesn't block tool execution on another. A crashed WhatsApp adapter doesn't take down Telegram.

**3. Observability is built into the runtime.** Connect to a running BEAM node with `:observer` or `remote_console` and you can inspect every process, every message queue, every supervision tree, every ETS table — live, in production. No logging framework, no metrics pipeline, no APM vendor. The runtime *is* the observability platform. For an AI agent system where understanding what the agent is doing (and why) is a safety requirement, this matters.

OpenClaw proved the product. Opal proved the architecture. Loom would combine both: the channel ecosystem and user-facing features of OpenClaw, running on a core that was designed from the ground up for the properties that matter most in an AI agent — fault isolation, hot upgrades, and runtime self-modification. The name fits: a loom runs on a beam, and weaves new patterns at runtime.

Five weeks of focused work. One architect. Unlimited tokens. The tools are all there.
