---
title: "CAMBRIAN: It Reproduces"
date: 2026-03-29
meta: "Agent Architecture"
series: "Self-Improving Agents"
series_part: 13
prev: "CAMBRIAN: The Loop Closes"
prev_url: 2026-03-27-cambrian-first-loop.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-29-cambrian-reproduces.html
---

# CAMBRIAN: It Reproduces

Two days ago, Gen-2 failed with 5 tests out of 59 and Opus was overloaded. Today, Gen-1 produced five viable offspring in 44 minutes. M1 is done.

---

## Two Spec Bugs, Not Code Bugs

The Part 12 failures — and every failed generation from Gen-2 through Gen-7 in the previous run — had a common root: the spec wasn't describing the environment precisely enough. We fixed two gaps.

**Streaming requirement.** The Anthropic SDK raises an error when you call `client.messages.create()` with `max_tokens=32768`. The spec said "call the API." It needed to say *how*: use `async with client.messages.stream(...) as stream` and `await stream.get_final_message()`. Every generation that used the non-streaming call hit a timeout-class error before it could even generate output.

**aiohttp test pattern.** Gen-2's 5 failing tests used `@unittest_run_loop`, a decorator removed in aiohttp 3.8+. The spec mentioned aiohttp as a dependency. It needed to show the correct fixture pattern: `aiohttp_client` from `pytest-aiohttp`, `aiohttp_server` for mock servers, and `asyncio_mode = "auto"` in pytest.ini. Once the spec included a concrete example, the LLM stopped generating the old pattern.

Both patches landed in CAMBRIAN-SPEC-005 v0.11.0 and in Gen-1's `SYSTEM_PROMPT`. After that, neither error class appeared again across any of the 8 subsequent generation attempts.

The lesson from Part 12 bears repeating: **the spec is a description of the environment, not the agent.** Architectural intent takes paragraphs. Environmental constraints take pages.

---

## The Run

Gen-1 ran autonomously for 44 minutes, producing 10 generations in total — 5 viable offspring out of 10 attempts.

*[Interactive generation tree — click any generation to see details]*

The pattern that emerges: every failed generation failed on exactly 1 test out of 58–69 passing. These aren't architectural failures. They're test-level mismatches where the LLM's generated test asserted subtly wrong behavior — always self-corrected on the next retry, with the failure diagnostics in context.

Two moments worth noting:

**Gen-4: parse repair.** Gen-4's LLM response had an unclosed `<file>` block. The parse repair loop caught it, sent the malformed response back to the LLM with an error message, received a corrected version, and the artifact was written with all 16 files intact. This mechanism — added to guard against truncated responses — worked exactly as designed on its first real invocation.

**Gen-6: retry with diagnostics.** Gen-5 failed with 1/58 tests. The retry prompt carried the exact failure — test name, assertion, actual vs. expected — to Gen-6. Gen-6 corrected it and passed 69/69 tests, the highest test count in the run. The informed retry isn't just "try again with the same prompt." It's "here's what broke, fix it."

---

## By the Numbers

- **474,834** cumulative tokens across all 10 generations
- **44 minutes** wall time
- **~$1.50** estimated cost (claude-sonnet-4-6 throughout)
- **50%** first-try success rate (5 promoted / 10 attempts)
- **100%** test pass rate on every promoted generation

The 50% success rate is higher than expected given that the retry mechanism exists specifically for the failure case. In practice, "fail → diagnose → succeed" in two attempts is the normal path, not a failure mode.

---

## What This Means

M1 was defined as: Prime reads spec → generates codebase → offspring passes test rig → offspring can do the same. Every promoted generation is a complete, working code factory. Gen-8, Gen-9, and Gen-10 are each capable of running this exact loop and producing the next generation. The chain is real.

It also means the two known blockers from Part 12 — streaming errors and deprecated test patterns — are eliminated by spec precision, not code sophistication. The LLM is capable of generating correct code when the spec tells it what "correct" means in this specific environment.

---

## What Comes Next

M1 proves that spec → code → verification → promotion is a stable loop. M2 asks the harder question: **what if the spec itself is the thing that evolves?**

Concretely, M2 Stage 1 involves:

- **Container isolation hardening.** Currently, the organism and test rig share a filesystem volume. Under evolutionary pressure, a sufficiently adversarial spec could describe code that writes fake viability reports. This gets closed before any population runs.

- **Fitness vectors.** Right now, viability is binary: 100% tests pass or it's rolled back. Real evolution needs gradients — test count, source lines, token efficiency, dependency minimalism. A 15-dimension fitness vector lets the system distinguish "barely viable" from "robustly viable."

- **Spec mutation with grammar constraints.** Unconstrained LLM mutation of a natural-language spec wastes attempts on syntactically or semantically broken variants. A formal spec grammar gates mutations to the valid space before evaluation.

- **Population campaigns.** Instead of one lineage evolving sequentially, run short parallel campaigns — evaluate 5–10 variants of a spec change, promote the fittest, discard the rest.

The research work is done: we studied A-Evolve (Amazon's agent evolution framework, open-sourced in early 2026), ran adversarial review of our assumptions, and updated the M2 proposal. The architecture maps to what A-Evolve calls "observe → evolve → gate → reload." The difference is we're evolving specs, not agent behavior — genotype mutation rather than phenotype tuning.

The honest caveat: reproduction is a solved problem. Evolution is harder. Selection pressure can optimize for the wrong things. Fitness functions can be gamed. Self-referential fitness — where the organism evolves to game its own fitness measure — is a real risk, not a theoretical one. The next post will have more to say about that when the first M2 campaign runs.

---

*Previous: [CAMBRIAN: The Loop Closes](2026-03-27-cambrian-first-loop.md)*
