---
title: "The program.md Protocol: Steering Self-Improvement with a Contract"
date: 2026-03-13
meta: "Agent Architecture"
series: "Self-Improving Agents"
series_part: 8
prev: "The Prime and the Lab"
prev_url: 2026-03-12-recursive-self-improvement.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-13-program-md.html
---

# The program.md Protocol: Steering Self-Improvement with a Contract

Yesterday we had an architecture — three components and a modification cycle. Today we refined how to steer it. The answer is a file: `program.md`. A collaboratively written contract between user and agent that defines what to build, how to test it, and what success looks like. This post describes the refined execution model, what changed from the original spec, and why the hardest problem isn't code modification — it's state continuity across generations.

The [previous post](2026-03-12-recursive-self-improvement.md) described a system where the Prime proposes modifications, tests them in a Lab, and promotes what works. That's the mechanism. But it left a critical question unanswered: **who decides what to improve?**

In Karpathy's *autoresearch*, the answer is a research program — a document that defines the experiment, the methodology, and the success criteria. The researcher writes it. The system executes it. We adopt the same pattern.

---

## What Changed

The original spec cast the Prime as both the brain and the worker. It proposed modifications, spawned Labs to test them, judged the results, and promoted or reverted. The user steered by chatting with the Prime and saying things like "make your read-file tool return line numbers."

This works for small, targeted changes. It fails when improvements require sustained, multi-step effort — building an LLM provider abstraction layer, redesigning the context manager, adding a new tool with tests. These aren't single-prompt tasks. They need a spec.

The refined model separates concerns:

| Role | Original spec | Refined spec |
|------|--------------|--------------|
| **User** | Chats with Prime, asks for changes | Collaborates with Prime to write `program.md` |
| **Prime** | Worker + judge | Orchestrator + judge |
| **Lab** | Passive eval server (~50 lines) | Full autonomous agent with LLM access |
| **Supervisor** | Container lifecycle + versions dir | Container lifecycle + git ops + timeout enforcement |

The key shift: **Labs are now full agents.** They run the complete agentic loop with LLM access. They don't just evaluate forms — they write code, run tests, and iterate autonomously. The Prime doesn't tell them how to implement the change. The `program.md` tells them what to achieve. The Lab figures out the how.

---

## The program.md Contract

The `program.md` is the sole directive for a Lab run. It defines three things:

- **Task spec** — what to build or improve
- **Acceptance criteria** — how to know it's done
- **Success conditions** — what Prime should verify before promoting

The user doesn't write it from scratch. The user describes what they want in conversation with Prime. Prime drafts the `program.md`, presents it for review, refines it based on feedback, and only spawns the Lab once the user approves. This is the control surface.

**The analogy:** `program.md` is to the Lab what a pull request description is to a code reviewer — it states the intent, the scope, and the definition of done. Except the "reviewer" is the Prime, and it doesn't just read the code. It runs the tests independently.

Acceptance criteria are heterogeneous:

| Criteria type | Example | Who checks |
|--------------|---------|------------|
| **Tests** | "All providers in `test/loom/llm_provider_test.cljs` pass" | Lab + Prime independently |
| **Eval probes** | "`/status` returns valid JSON with `:ready true`" | Prime sends probes |
| **Benchmarks** | "Round-trip latency under 200ms" | Prime measures |

Prime evaluates using option C: **trust but verify.** The Lab runs its own tests as part of its work. Prime independently re-runs them and adds its own probes. Both must pass for promotion.

---

## The Execution Flow

Eight steps, from conversation to promotion:

1. **Discuss.** User describes the improvement to Prime.
2. **Draft.** Prime generates `program.md` with task spec and acceptance criteria.
3. **Approve.** User reviews, refines, approves.
4. **Spawn.** Supervisor clones repo onto `lab/gen-N` branch, boots Lab container on `loom-net`.
5. **Execute.** Lab reads `program.md`, works autonomously. Commits before executing risky changes.
6. **Verify.** Prime re-runs tests, sends eval probes, checks benchmarks independently.
7. **Promote.** Supervisor merges branch into `main`, tags `gen-N`, serializes Prime state.
8. **Restart.** New Prime boots from promoted code with loaded state.

Boot is the "go" signal. No coordination protocol needed. The Supervisor creates the container with the repo clone and `program.md` already in place. When it starts, the Lab reads `program.md` and begins. Prime polls `/status` until done or the 5-minute timeout fires.

---

## Networking

Apple Containerization on macOS 26 provides `vmnet`-backed virtual networking with built-in DNS:

```bash
container network create loom-net
container run --name lab-gen-1 --network loom-net ...

# Prime reaches Lab by name
curl http://lab-gen-1:8402/status
```

No port mapping. No IP discovery. The container name is the address.

---

## Git as the Ledger

Every generation is a branch. Every promotion is a merge and a tag.

```
main              ———*———————*———————*——→
                      |           |           |
                 tag: gen-0  tag: gen-1  tag: gen-2
```

`main` always tracks known-good code. Tags provide unlimited rollback depth — not just N-1, but any previous generation.

The Supervisor maintains `generations.edn` — a ledger with metadata per generation: parent, branch, program.md hash, outcome, timestamps, container ID. This file is versioned in git. The full lineage is recoverable from repo history alone.

---

## The Hard Problem: State

Code promotion is straightforward — merge a branch. But a running agent has state: conversation history, task context, accumulated knowledge. When Gen N promotes to Gen N+1, the new Prime starts fresh. How does it know what Gen N knew?

Three sub-problems:

1. **Serialize** — snapshot outgoing state as EDN
2. **Load** — hydrate incoming Prime from snapshot
3. **Evolve** — handle schema changes between generations

Problem 3 is where it gets interesting. If Gen N stores history as a flat vector and Gen N+1 restructures it as a tree, the snapshot is useless. Schema migration in a self-modifying system is a known-hard problem.

**v0 approach:** state as EDN with a version tag. If schema doesn't match, start fresh. Git tags preserve old state. Proper migration deferred until state format stabilizes.

---

## What's Next

Phase 1a (self-hosted ClojureScript eval) is done — 9 tests, 17 assertions passing. The implementation plan has six phases remaining, each a falsifiable experiment. Next: boot ClojureScript in an Apple container and measure startup time.

**The refined bet:** the hardest part isn't getting the agent to modify code. It's getting the cycle right — the handoff from user intent to `program.md` to autonomous Lab to verified promotion. Get the cycle right and the modifications follow.

---

## References

- [The Prime and the Lab](2026-03-12-recursive-self-improvement.md) — Original architecture spec. This post refines the execution model.
- [The Autoresearch Pattern](2026-03-11-self-improving-agents.md) — Karpathy's blueprint. The `program.md` pattern derives from this.
- [Apple Containerization](https://github.com/apple/container) — VM-per-container with custom network DNS.
- [Malli](https://github.com/metosin/malli) — Schema library for the fixed-point contracts.
- [Pi Coding Agent](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) — Radical minimalism. Our Lab inherits this philosophy.
