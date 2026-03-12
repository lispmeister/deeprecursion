---
title: "The Prime and the Lab: Recursive Self-Improvement for Coding Agents"
date: 2026-03-12
meta: "Agent Architecture"
series: "OpenClaw/Loom"
series_part: 7
prev: "Loom in Lean: Bootstrapping a Verified Self-Improving Agent"
prev_url: 2026-03-11-loom-in-lean.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-12-recursive-self-improvement.html
---

# The Prime and the Lab: Recursive Self-Improvement for Coding Agents

We abandoned Lean. We abandoned the BEAM. [ByteRover](https://clawhub.ai/byteroverinc/byterover) solved memory. What's left is the real question: can an agent rewrite its own code, test the result in isolation, and keep only what works? This post is both the argument and the spec. A ClojureScript agent running in a container, modifying itself, and proving the modification in a second container before promoting it. No frameworks. No build step. Just a loop, a contract, and two containers.

The [previous post](2026-03-11-loom-in-lean.md) proposed Lean as the verification layer for self-modifying agents, with memory as the beachhead organ. Then [ByteRover](https://clawhub.ai/byteroverinc/byterover) shipped a production-ready memory system — hierarchical context trees, automatic curation, daily knowledge mining — and the entire Lean+QMD approach became unnecessary.

Good. That failure clarified what we're actually trying to build. Not a verified memory system. Not a formal proof engine. Something simpler and more dangerous: **an agent that can rewrite its own code, test the rewrite in isolation, and promote it if it works.**

This post is the blueprint.

---

## Why ClojureScript

We considered two languages for self-modifying agent code: ClojureScript and Elixir. The [earlier posts](2026-03-05-runtime-self-modification.md) made a strong case for the BEAM — supervision trees, hot code swapping, process isolation. But those properties solve the wrong problem. We don't need a runtime that catches crashes. We need a language where *code is data the agent can manipulate*.

ClojureScript is a Lisp. Code is represented as data structures — lists, vectors, maps — the same structures the agent reasons about. The [S-expressions post](2026-02-13-sexp-native-editing.md) argued that LLMs should think in homoiconic representations. ClojureScript *is* that representation.

The self-hosted compiler (`cljs.js`) seals it. A ClojureScript program can compile and evaluate ClojureScript at runtime without a build step, without a JVM, without anything beyond Node.js. The agent can `eval-str` a modified function and watch what happens. No compilation pipeline. No deploy cycle. Just data in, behavior out.

And ClojureScript interoperates with JavaScript natively. OpenClaw is 57,000 lines of TypeScript. No bridge protocol needed — ClojureScript calls JS functions directly. This is the difference between growing an organ inside the patient and transplanting one from another species.

| Property | ClojureScript | Elixir |
|---|---|---|
| Code is data | Yes (homoiconic) | No |
| Runtime eval | `cljs.js/eval-str` | `Code.compile_string` |
| OpenClaw integration | Native JS interop | Bridge required |
| Isolation | None built-in | Process-level |
| Rollback | None built-in | Supervisor trees |
| Build step | None (self-hosted) | Mix release |

ClojureScript's weakness is obvious: no isolation, no rollback, no supervision. A bad `eval-str` corrupts the runtime. That's why the agent doesn't modify itself in place. It modifies a copy. In a container.

---

## The Architecture

Three components, all in ClojureScript on Node.js.

**Prime Container.** The live agent — the known-good version that talks to Claude, executes tools, and does real work. When it proposes a modification to its own code, it doesn't apply it. It sends the proposal to the Supervisor.

**Supervisor.** Runs on the host (macOS, not in a container). Its only job is container lifecycle: it takes the Prime's source code, applies the proposed modification, and boots a Lab Container with the result. The Supervisor doesn't evaluate code. It doesn't talk to Claude. It creates, starts, stops, and destroys containers.

**Lab Container.** Ephemeral. It boots the modified agent code and exposes a form evaluation server — a lightweight TCP service that accepts ClojureScript forms as EDN, runs `cljs.js/eval-str`, and returns results. The Prime connects to this server and probes the modified code: call functions, inspect state, run test scenarios. The Lab doesn't know it's being tested. It's just running.

If the Prime is satisfied, it tells the Supervisor to promote: the Lab's source becomes the new Prime, the old Prime is archived, and the system restarts with the improved version. If the Prime isn't satisfied, the Supervisor destroys the Lab. Nothing changes.

---

## The Modification Cycle

Every self-modification follows six steps:

1. **Propose.** The agent identifies something to improve, generates modified `.cljs` source files, and POSTs a proposal to the Supervisor with a description, the modified files, and a rationale.

2. **Spawn Lab.** The Supervisor copies the current Prime source, applies the modifications, and boots a Lab container. The Lab starts: Node.js, self-hosted ClojureScript compilation, eval server listening. Boot time: ~3–5 seconds.

3. **Probe.** The Prime connects to the Lab's eval server via TCP. It sends ClojureScript forms as EDN strings: function calls, state inspections, test scenarios. Each probe returns a result.

4. **Judge.** The Prime evaluates the probe results. For v0: did all probes return `:ok`? No crashes, no errors, no timeouts. The Prime constructs a Verdict — `:promote` or `:revert` — with evidence and reasoning.

5. **Promote or Revert.** If `:promote` — the Supervisor copies the Lab's source to `versions/N+1/`, stops the current Prime, restarts with the new version. If `:revert` — the Supervisor destroys the Lab. Nothing changes.

6. **Repeat.** The cycle completes. The agent can immediately propose another modification, building on success or trying a different approach after failure.

The cycle is the [autoresearch pattern](2026-03-11-self-improving-agents.md) made concrete. The three separations hold: the **judge** (Prime probing the Lab) is separate from the **subject** (Lab running modified code), which is separate from the **process** (Supervisor managing containers). The keep/revert boundary is a container wall.

---

## The Form Evaluation Server

The Lab doesn't run a full nREPL. It runs something simpler: a TCP server that accepts EDN forms and evaluates them. About 50 lines of ClojureScript.

```clojure
;; Lightweight form evaluation server
;; Lab container exposes this on a known port

(ns lab.eval-server
  (:require [cljs.js :as cljs]
            [cljs.reader :as reader]
            ["net" :as net]))

(def compiler-state (cljs/empty-state))

(defn eval-form [form-str callback]
  (cljs/eval-str
    compiler-state
    form-str
    "lab-eval"
    {:eval cljs/js-eval :context :expr}
    (fn [{:keys [value error]}]
      (callback (pr-str (if error
                          {:status :error :message (.-message error)}
                          {:status :ok    :value value}))))))
```

The Prime connects as a TCP client, sends forms, gets results as EDN. [Malli](https://github.com/metosin/malli) validates the protocol — every request and response conforms to a schema. The schemas themselves are EDN — data the agent can inspect and eventually modify.

```clojure
;; Malli schemas for the eval protocol

(def EvalRequest
  [:map
   [:form :string]
   [:timeout {:optional true} :int]])

(def EvalResponse
  [:map
   [:status [:enum :ok :error]]
   [:value {:optional true} :any]
   [:message {:optional true} :string]])
```

---

## The Five Tools

The agent has five tools. Four are borrowed from [Mario Zechner's Pi coding agent](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/), which proved that four tools and a 1,000-token system prompt can match elaborate frameworks on Terminal-Bench.

| Tool | What it does |
|---|---|
| `read-file` | Read a file's contents |
| `write-file` | Write content to a file |
| `edit-file` | Replace a string in a file |
| `bash` | Execute a shell command |
| `self-modify` | Propose a modification to the agent's own source |

No MCP. Pi's lesson: a 21-tool MCP server consumes 13,700 tokens before the agent does anything.

---

## Containers as the Keep/Revert Boundary

[Apple Containerization](https://github.com/apple/container) runs each container as a dedicated lightweight VM via the Virtualization framework. Not shared-kernel containers — full VM isolation with sub-second boot times. A bad modification can't escape. A malicious `eval-str` can't touch the host.

This is the safety layer that ClojureScript lacks. Instead of building isolation into the language (Elixir's approach) or verification into the type system (Lean's approach), we put the entire experiment in a box. The box is disposable.

The Supervisor maintains a `versions/` directory:

```
versions/
  v001/          ← initial agent source
    src/
      agent/
        core.cljs
        tools.cljs
        loop.cljs
  v002/          ← first successful modification
    src/
      agent/
        core.cljs
        tools.cljs
        loop.cljs   ← modified
```

Promotion copies the Lab's source to `versions/N+1/`. Revert restarts Prime at `versions/N`. Every version is preserved.

---

## The Contract

Communication between components follows a fixed schema, enforced by [Malli](https://github.com/metosin/malli). The [Wezzard agentic loop](https://wezzard.com/post/2025/09/build-your-first-agentic-loop-9d22) demonstrated that contract-driven prompts turn free-form LLM output into predictable, machine-readable communication.

```clojure
;; Modification proposal: Prime → Supervisor
(def Proposal
  [:map
   [:id :string]
   [:description :string]
   [:files [:vector [:map [:path :string] [:content :string]]]]
   [:rationale :string]
   [:parent-version :int]])

;; Verdict: Prime → Supervisor
(def Verdict
  [:map
   [:proposal-id :string]
   [:decision [:enum :promote :revert]]
   [:evidence [:vector ProbeResult]]
   [:reasoning :string]])
```

The contract is the fixed point. The agent can modify its tools, its loop, its system prompt, its probe strategy — but not the contract schemas. This is the [autoresearch pattern's](2026-03-11-self-improving-agents.md) "well-chosen fixed point."

---

## What Can the Agent Modify?

Everything except the contract:

- **The agentic loop** — how it processes messages and dispatches tools
- **The system prompt** — what instructions it gives Claude
- **Tool implementations** — how read, write, edit, bash, self-modify work
- **The probe strategy** — how it tests modifications in the Lab
- **New tools** — the agent can add tools that don't exist yet
- **The eval server** — how the Lab accepts and processes forms

This is full self-modification. The only constraints are the Malli schemas (communication protocol) and the container boundary (isolation layer).

---

## Observability

Both the Prime and Supervisor expose HTTP endpoints:

| Endpoint | Component | Returns |
|---|---|---|
| `GET /` | Both | Dashboard (single HTML page) |
| `GET /logs` | Both | SSE stream of events |
| `GET /stats` | Both | JSON: counts, uptime, current version |
| `POST /chat` | Prime | Send a message to the agent |
| `GET /lab/repl` | Supervisor | Proxied view of Lab's eval session |
| `GET /versions` | Supervisor | Version history with diffs |

The user interacts with the agent through the Prime's `/chat` endpoint. Pi's lesson applies: **observability over automation**.

---

## What Kills This

1. **Self-hosted ClojureScript is slow.** Boot time could be 5–10 seconds. Modification cycle: ~15 seconds end-to-end. Only 4 tries per minute.

2. **"Doesn't crash" isn't a judge.** Subtle degradation passes the v0 test every time. Real judgment requires real metrics — that's v1.

3. **The probe strategy problem.** The agent can modify its own probe strategy to be less rigorous, drifting toward accepting bad changes.

4. **Apple Containerization is pre-1.0.** Breaking changes between minor versions. [UTM](https://docs.getutm.app/) is the fallback.

5. **Context window economics.** Running this loop 24/7 at Claude's pricing could cost hundreds per day.

---

## The MVP

Six components, all ClojureScript, no external dependencies beyond Node.js and the `container` CLI.

| Component | Description | Runs on |
|---|---|---|
| Shared library | Malli schemas, eval protocol, HTTP helpers | All |
| Supervisor | Container lifecycle, version management, dashboard | Host |
| Agent loop | Claude API, 5 tools, agentic loop | Prime |
| Eval server | TCP form evaluation, ~50 lines | Lab |
| OCI image | Node.js + self-hosted ClojureScript runtime | Containers |
| Dashboard | Single HTML page, SSE logs, stats, chat | Served by Supervisor & Prime |

**Not in v0:** streaming, multi-provider, MCP, ByteRover, TUI, sub-agents, formal verification, autonomous triggers.

---

## What Comes Next

Build it. The sequence:

1. **OCI image.** Node.js + self-hosted ClojureScript. Verify `cljs.js/eval-str` works in a container. Measure boot time.
2. **Eval server.** TCP server, 50 lines. Verify round-trip.
3. **Supervisor.** Shell out to `container` CLI. Version directory management.
4. **Agent loop.** Claude API client, 5 tools, main loop.
5. **Self-modify tool.** Generate proposal, send to Supervisor, probe Lab, judge result.
6. **First modification.** The agent changes one of its own functions, tests it, promotes it. That's the proof.

Each step is a falsifiable experiment. A working prototype is realistic in two to three weeks.

**The bet:** one successful self-modification — an agent that rewrites one of its own functions, tests the result in a Lab, and promotes the improvement — is worth more than this entire series. Ship it.

---

## References

- [ByteRover](https://clawhub.ai/byteroverinc/byterover) — Long-term memory infrastructure for AI agents
- [ByteRover OpenClaw integration](https://docs.byterover.dev/autonomous-agents/openclaw) — Memory flush, knowledge mining, context enrichment
- [Apple Containerization](https://github.com/apple/container) — Lightweight Linux containers as VMs on macOS 26
- [UTM](https://docs.getutm.app/) — Scriptable VM management for macOS
- [Pi Coding Agent](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) (Mario Zechner) — Radical minimalism in agent design
- [Build Your First 24/7 Agentic Loop](https://wezzard.com/post/2025/09/build-your-first-agentic-loop-9d22) (Wezzard) — Contract-driven evaluator/executor architecture
- [Malli](https://github.com/metosin/malli) (Metosin) — Data-driven schema library for Clojure/ClojureScript
- [ClojureScript Self-Hosting Guide](https://clojurescript.org/guides/self-hosting) — Runtime compilation and evaluation
