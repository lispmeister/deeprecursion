---
title: "It Rewrote Itself: Loom's First Autonomous Self-Modification"
date: 2026-03-17
meta: "Agent Architecture"
series: "Self-Improving Agents"
series_part: 10
prev: "First Light: Loom's Self-Modification Pipeline in 2,214 Lines"
prev_url: 2026-03-16-loom-mvp.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-17-it-rewrote-itself.html
---

# It Rewrote Itself: Loom's First Autonomous Self-Modification

Yesterday we had a pipeline. Today it did something with it.

At 19:24 UTC on March 17, 2026, a ClojureScript agent running inside an Apple Containerization VM opened its own source code, decided two private functions should be public, wrote 63 lines of tests to prove they work, and committed. A second agent — running outside the container — pulled the branch, ran 236 tests, asked an LLM to review the diff, and promoted the change to master. No human touched a file. The whole thing took 56 seconds.

This is gen-72. It is the first time Loom autonomously modified its own agent code, verified the modification, and promoted it. But the real story isn't the 56 seconds that worked. It's the 16 generations that didn't.

---

## The Bet

Five posts ago we made a specific claim: an agent that rewrites one of its own functions, tests the result in a Lab container, and promotes the improvement is worth more than the entire blog series that described it. The [MVP post](2026-03-16-loom-mvp.md) proved the pipeline ran — containers booted, branches propagated, tests executed — but every successful generation was a trivial file operation. Add a comment. Create a text file. Count source files.

We hadn't proven the thing that matters: can the agent change *itself*?

---

## Sixteen Failures

We pointed the autonomous loop at the codebase and pressed go. Five runs. Three models. Sixteen generations before the first promotion.

| Run | Model | Gens | Promoted | What Happened |
|-----|-------|------|----------|---------------|
| 1 | Haiku | 5 | 0 | Race condition: agent sees "done" before supervisor fetches the branch |
| 2 | Haiku | 5 | 0 | Gen-58 passes all tests — LLM reviewer rejects it anyway |
| 3 | Anthropic | 3 | 0 | API credits exhausted mid-run |
| 4 | Minimax M2.5 | 3 | 0 | Model reads every file in the repo. Writes nothing. |
| 5 | **Opus 4.6** | **1** | **1** | **56 seconds. Clean. First try.** |

Every failure taught us something we couldn't have learned from unit tests.

---

## The Six Bugs That Had to Die

The pipeline looked correct. It passed 142 tests. It had run 17 stable generations of trivial tasks. Then we asked it to do real work autonomously, and six bugs crawled out of the walls.

**1. The race condition.** The agent polls the Lab container directly — `GET /status` — and sees "done." It immediately tries to verify by checking out the Lab's branch. But the supervisor hasn't finished `git fetch` from the container's repo clone yet. `git checkout lab/gen-47` fails: "pathspec did not match any file(s) known to git." The branch exists inside the container. It doesn't exist on the host yet.

Fix: retry loop. Three attempts, two seconds apart. The fetch completes in the gap.

**2. Local changes blocking checkout.** The agent has been modifying `self_modify.cljs` — the same file the Lab changed. `git checkout lab/gen-54` refuses: "Your local changes to the following files would be overwritten." The pipeline never considered that someone might be editing the codebase while it runs.

Fix: `git stash --include-untracked` before checkout, `git stash pop` after returning to master.

**3. .gitignore duplication.** Every Lab spawn appends `lab-worker.js` to `.gitignore`. After 50 generations, that's 50 identical lines. Worse: the diff now includes .gitignore changes in every generation, confusing the LLM reviewer.

Fix: check before appending. One line of regex.

**4. program.md gitignored.** We added `program.md` to `.gitignore` so it wouldn't leak into master. But when .gitignore already had the entry and nothing else changed, `git commit` failed with "nothing to commit" — the Lab's task spec vanished from history.

Fix: stop gitignoring `program.md`. It belongs in the Lab branch. It gets removed during promotion.

**5. The LLM reviewer was too strict.** Gen-58 (Haiku) passed all tests. The code change was valid — it trimmed a tool description to save tokens. The LLM reviewer rejected it because the diff included .gitignore changes and a container-relative path (`/workspace/`). Both are normal Lab artifacts, not bugs.

Fix: updated the review prompt. "IGNORE: changes to .gitignore, minor style differences, hard-coded paths like /workspace/ (Labs run in containers)."

**6. Timeout handler forgot to fetch.** When a Lab times out, the supervisor killed the container but never ran `git fetch` on the Lab's branch. The branch existed in the container's clone but was never copied to the host. Any subsequent verify attempt failed silently.

Fix: fetch on all outcomes — done, failed, and timed out.

None of these bugs showed up in unit tests. They required the full pipeline running against a real LLM, a real container, and real concurrent git operations. This is why you run the thing.

---

## The Model Gap

We tested three models as Lab workers. The results are not subtle.

**Haiku ($0.10/generation)** gets close. It reads the right files, understands the task, writes code that mostly works. But "mostly" doesn't survive two-stage verification. Out of 10 Haiku generations, zero promoted. It's a useful prototyping model — cheap enough to iterate on `program.md` design — but it can't land the plane.

**Minimax M2.5 ($0.05/generation, coding plan)** exhibits a behavior we haven't seen documented elsewhere: it reads. And reads. And reads. It opens every file the task references. It opens files the task doesn't reference. It reads the same file twice. Then it reports done, having written nothing. Three generations, three empty branches. The pipeline handled it correctly — verification caught the no-op — but the model simply doesn't write code. This isn't a pipeline bug. It's a model-level behavior.

**Opus 4.6 ($1-2/generation)** did it on the first try. 56 seconds from spawn to done. It made the right architectural call — `defn-` to `defn` for testability, since ClojureScript doesn't support `#'` var access for private functions — wrote 18 test assertions across 6 test functions, and committed. The LLM reviewer approved with high confidence. No retries. No partial work. The entire generation cost less than a cup of bad coffee.

The lesson: model capability is the binding constraint for autonomous self-modification, not infrastructure. The pipeline is the same for all three models. Only one produces promotable code.

---

## What Gen-72 Actually Changed

The task was deliberately chosen for safety: add unit tests for two pure helper functions that already existed in `self_modify.cljs`. No new features, no refactors, no behavioral changes. Just tests.

```
 program.md                              | 32 +++++++++++++++
 src/loom/agent/self_modify.cljs         |  4 +--
 test/loom/self_modify_helpers_test.cljs | 63 +++++++++++++++++++++++++++++
 3 files changed, 97 insertions(+), 2 deletions(-)
```

The Lab made two changes to existing code (`defn-` → `defn` on two functions) and created a new test file. The test file covers:

- `parse-test-counts` — parses "Ran 42 tests containing 100 assertions.\n0 failures, 0 errors." into structured data
- `parse-shortstat` — parses "3 files changed, 37 insertions(+), 11 deletions(-)" into structured data

Both are pure functions. No side effects, no mocking, no async. The ideal first target for autonomous modification: low risk, high verifiability, measurable improvement (test count went from 230 to 236, assertions from 589 to 607).

It's not impressive as a code change. It *is* impressive as a proof. The agent read its own source, understood the function signatures, understood why the functions were private and why that matters in ClojureScript, made the right call to change visibility, and wrote tests that exercise edge cases (passing output, failing output, garbage input, missing fields). An agent reasoned about its own code and improved it.

---

## Two-Stage Verification

Gen-72 had to pass two independent checks before promotion.

**Stage 1: Tests.** Prime checks out `lab/gen-72`, runs `npm test`, confirms all 236 tests pass with 607 assertions and zero failures. This catches regressions — the Lab's changes didn't break anything that already worked.

**Stage 2: LLM Review.** Prime sends the full diff to an LLM with instructions to evaluate: Does this change do what program.md asked? Does it introduce bugs, vulnerabilities, or regressions? Is the code quality acceptable? The reviewer returns APPROVED or REJECTED with a confidence level and reasoning.

Gen-72: APPROVED, high confidence.

This two-stage approach caught something interesting during earlier runs. Haiku gen-58 *passed all tests* — zero failures, zero errors. The code change was valid (trimming a tool description). But the LLM reviewer rejected it because the diff was cluttered with .gitignore artifacts. That was a false negative — the reviewer was too strict — but it demonstrated the value of the second stage. Tests measure correctness. The LLM measures intent.

---

## The Numbers

| Metric | MVP (Mar 16) | Today (Mar 17) |
|--------|-------------|-----------------|
| Source files | 19 (2,214 LOC) | 26 (13,503 LOC) |
| Test files | 13 | 21 |
| Tests / Assertions | 90 / — | 236 / 607 |
| Generations run | 17 | 72 |
| Autonomous promotions | 0 | 1 |
| Infrastructure bugs found | 5 | 11 |
| Models tested as Lab workers | 1 (Haiku) | 3 (Haiku, M2.5, Opus) |
| Beads issues closed | — | 131 of 144 |

The codebase has grown 6x since the MVP — mostly test infrastructure, the autonomous loop driver, the reflect step, fitness scoring, and multi-provider support. The test suite has grown from 90 to 236. Every new component was tested before it was deployed, and then tested again by the autonomous loop breaking it in ways we didn't anticipate.

---

## Multi-Provider Support

One discovery from the autonomous runs: you want different models for different roles.

- **Prime** (reflect, verify, review) runs Sonnet. It needs to be smart but not expensive — it's making judgments, not writing code.
- **Lab** (autonomous code generation) runs Opus. It needs to be precise. A $1 generation that promotes is cheaper than ten $0.10 generations that don't.

Loom now supports split-provider configuration via `.env`:

```bash
# Prime uses Anthropic Sonnet
ANTHROPIC_API_KEY=sk-ant-...
LOOM_MODEL=claude-sonnet-4-20250514

# Lab uses Opus
LOOM_LAB_MODEL=claude-opus-4-6

# Or use a completely different provider for Labs
# LOOM_LAB_API_KEY=sk-cp-...
# LOOM_LAB_API_BASE=https://api.minimax.io/anthropic
# LOOM_LAB_MODEL=MiniMax-M2.5
```

Lab inherits from Prime unless overridden. This means you can experiment with different Lab models without touching the Prime configuration — and without risking your orchestration layer.

---

## What This Costs

We built a credit check script (`scripts/check-credits.sh`) after burning through API credits mid-run. It sends a 1-token probe to the API, verifies the key works, and estimates the cost of an autonomous run.

| Model | Cost/Generation | Success Rate | Effective Cost/Promotion |
|-------|----------------|--------------|--------------------------|
| Haiku | ~$0.10 | 0% (0/10) | — |
| Minimax M2.5 | ~$0.05 | 0% (0/3) | — |
| Opus 4.6 | ~$1-2 | 100% (1/1) | ~$1-2 |

One data point isn't a rate. But the pattern is clear: cheap models that can't promote are more expensive than expensive models that can. A 5-generation Opus run costs ~$5-10. A 5-generation Haiku run costs ~$0.50 and produces nothing.

The real cost equation includes Prime overhead: Sonnet runs reflect + review at ~$0.50/generation regardless of Lab model. So the total per-generation cost for an Opus Lab run is ~$2-3.

---

## Where We Stand

The pipeline works. Not "works in tests" — works against real models, real containers, real concurrent git operations, after surviving 16 failures that each hardened a different component.

**Proven:**
- Autonomous spawn → execute → verify → promote cycle
- Two-stage verification catches both test failures and semantic issues
- Multi-provider support (tested with 3 providers)
- Fitness scoring and generation reports
- CLI workflow (`node out/agent.js spawn/verify/promote`)

**Not yet proven:**
- Multiple consecutive autonomous promotions (we have 1)
- Reflect-driven task selection (reflect is implemented but not wired to the autonomous loop end-to-end)
- Fitness plateau detection in practice
- Tasks more complex than "add tests for pure functions"

---

## What's Next

The bottleneck has shifted. It's no longer infrastructure — that's battle-tested. It's no longer "does it work" — it does. The questions now are:

**Task selection.** Gen-72 was carefully chosen: pure functions, no side effects, low blast radius. What's the next rung? More test coverage is safe and measurable. Small refactors (extract helpers, rename internals) are riskier but more valuable. New features are the end goal but require precise `program.md` specs that the current reflect step may not produce reliably.

**Cost efficiency.** Opus works but isn't cheap. Can Sonnet 4 (when released) hit the sweet spot? Can we improve `program.md` quality enough for Haiku to succeed on simple tasks? The model gap is the main cost lever.

**Fitness gaming.** The current fitness function rewards test count: `score = (tests-run * 10) + assertions - (tokens / 1000)`. An autonomous agent optimizing this could write `(is (= 1 1))` a thousand times and claim improvement. The LLM reviewer is a partial defense — it would probably catch egregious gaming — but we haven't tested adversarial scenarios yet.

**The compelling demo.** "I pointed it at itself and walked away. When I came back, it had improved." Gen-72 is one step short of this. The loop ran once. We need it to run five times, each building on the last, with the reflect step choosing each task. That's the demo.

---

## The Lesson

Building a self-modifying agent is not one problem. It's a sequence of failures that each reveal a different assumption.

We assumed branch fetching was synchronous. It isn't. We assumed `.gitignore` management was idempotent. It wasn't. We assumed an LLM reviewer would focus on code quality. It fixated on formatting artifacts. We assumed any model could write code. One of them just reads.

Each of these assumptions was invisible in the spec, invisible in unit tests, and immediately obvious in production. The 16 failed generations before gen-72 weren't wasted work. They were the work. The pipeline that promoted gen-72 is fundamentally different from the pipeline that attempted gen-43 — six bugs fewer, three models tested, and a two-stage verification system tuned by real false negatives.

Self-modification isn't a feature you ship. It's a capability you earn by running the loop until the loop survives.

Gen-72 was 56 seconds. Getting there took five days.

---

## References

- [First Light: Loom's Self-Modification Pipeline in 2,214 Lines](2026-03-16-loom-mvp.md) — The MVP field report
- [The program.md Protocol](2026-03-13-program-md.md) — The steering mechanism for Lab runs
- [The Prime and the Lab](2026-03-12-recursive-self-improvement.md) — Original architecture spec
- [The Autoresearch Pattern](2026-03-11-self-improving-agents.md) — Karpathy's blueprint
- [Apple Containerization](https://github.com/apple/container) — VM-per-container on macOS 26
- [Pi Coding Agent](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) — Radical minimalism in agent design

---

*Previous: [First Light: Loom's Self-Modification Pipeline in 2,214 Lines](2026-03-16-loom-mvp.md)*
