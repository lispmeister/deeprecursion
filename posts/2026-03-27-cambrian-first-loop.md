---
title: "CAMBRIAN: The Loop Closes"
date: 2026-03-27
meta: "Agent Architecture"
series: "Self-Improving Agents"
series_part: 12
prev: "CAMBRIAN: What If the Spec Is the Organism?"
prev_url: 2026-03-18-cambrian.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-27-cambrian-first-loop.html
---

# CAMBRIAN: The Loop Closes

Nine days ago I wrote that Phase 1 — proving a spec can regenerate a working agent — would take a week and cost $20–50. Today, Gen-1 ran its generation loop for the first time. It generated Gen-2, submitted it to the test rig, got back a failure report, and rolled it back. All autonomously. The loop works.

It also cost more than $50 and took longer than a week. Here's what that bought us.

---

## What We Thought Phase 1 Was

The March 18 plan: write a spec, hand it to an LLM, watch it generate a working agent, measure costs, iterate. Simple.

What we underestimated: how much precise environmental knowledge has to be encoded in the spec before an LLM can reliably produce code that actually runs. Not code that looks correct. Code that compiles, passes tests, and survives a mechanical verification pipeline in a Python 3.14 Docker container.

Every generation from Gen-2 through Gen-7 died on the same class of bug before we even got the loop running correctly. The LLM wrote test strings containing unescaped newlines inside single-quoted string literals — legal in Python ≤3.11, a `SyntaxError` in 3.14. The spec said "Python 3.14 compatible." That's not enough. The spec needs to say *what Python 3.14 enforces* that earlier versions didn't.

This is the lesson Phase 1 taught: **the spec is not a description of the agent; it's a description of the environment the agent must survive in.** The architecture takes three paragraphs. The environmental constraints take three pages.

---

## What Actually Happened

**March 21.** Language choice: Python. Not because it's ideal — but because LLM coding accuracy correlates with training data density, and Python has more of it than anything else. Python benchmarks at ~93% on standard coding problems; Rust at ~85%; Elixir at ~70%. For M1, the right language is the one the LLM gets right, not the one with the best runtime properties.

**March 24.** Infrastructure: Supervisor (manages container lifecycle, generation history), Test Rig (mechanical verifier: build → test → start → health → report), Docker base image. Gen-0, a hand-crafted test artifact, validated the pipeline before any LLM involvement. This is the principle we're glad we followed: *verify the environment before testing the organism*. If the environment is broken, every organism will fail and the failures will be misleading.

**March 24–26.** Gen-1 required three iterations to pass the test rig. The bugs found along the way weren't trivial: the `/stats generation` field has to return the artifact's *own* generation number (set from an env var at startup), not the next generation the loop is about to produce — those diverge immediately in any real deployment. Tests run before the server starts, so they can't hit localhost; they need an in-process test client. The Docker socket on macOS is not at `/var/run/docker.sock` when using Docker Desktop — it's at `~/.docker/run/docker.sock` and `aiodocker` won't follow the symlink.

These aren't hard problems. They're the kind of problems you only find by running the thing.

**March 26.** Gen-1 promoted. All 5 pipeline stages passing, all 3 contracts passing.

**March 27 (morning).** First autonomous run attempt. Gen-2 through Gen-7 all failed with Python 3.14 syntax errors in the test strings for the file parser. The LLM understands the rule when prompted — it just doesn't apply it reliably while generating 400 lines of code. Sonnet 4.6 failed this consistently across all retries. An Opus 4.6 probe (5 attempts) passed 4/4.

Two other bugs found this morning: the parser itself was broken. The original `parse_files()` used a dotall regex that stopped at the first `</file>` tag — which is also what the LLM generates *inside test fixture strings*. Even a perfectly generated `test_generate.py` would be silently truncated by Prime's own extraction step. Fixed by changing the closing delimiter to `</file:end>` (the colon makes it impossible to appear in natural content) and replacing the regex with a 20-line state machine.

**March 27 (afternoon).** Spec updated to v0.11.0. Gen-1 code brought up to spec. The loop ran.

---

## The Loop

At 12:14:35, Gen-1 started inside a Docker container with the artifacts root mounted at `/workspace`. It served `/health` and `/stats` on port 8401 and launched its generation loop.

At 12:14:35, it called the Anthropic API with the spec and generation history as context. Sonnet 4.6, 11,553 input tokens.

At 12:17:21 — two minutes and 46 seconds later — it received a response: 15,125 output tokens, a complete Gen-2 codebase, 12 files. It parsed the files, wrote them to disk, built the manifest, and called `POST /spawn` on the Supervisor.

The Supervisor created a `gen-2` branch in the artifacts repo, started a test rig container with Gen-2 mounted as `/workspace`, and returned `{"ok": true}`.

At 12:17:21, the test rig started. Twelve seconds later:

- **Manifest:** ✓
- **Build:** ✓ (12,346ms — installing dependencies from scratch)
- **Test:** ✗ — 5 of 59 tests failed

Gen-2 used `@unittest_run_loop`, a decorator deprecated in aiohttp 3.8+ that's unnecessary when `pytest-asyncio` runs in AUTO mode. The LLM generated tests correctly structured for an older async testing pattern. 54 tests passed. 5 failed.

Gen-1 received the viability report. Status: non-viable. It called `/rollback`. The Supervisor tagged the artifact `gen-2-failed-3` and deleted the branch. Gen-1 built a retry prompt with the full diagnostics and escalated to Opus 4.6.

Opus returned HTTP 529: overloaded. Three times in a row. Retries exhausted. Generation loop stopped. Gen-1 kept serving `/health`.

Total tokens for the loop: 26,678. Estimated cost: ~$0.08.

---

## What The Loop Got Right

The loop mechanics are correct. Every step worked as designed:

- Gen-1 read generation history from the Supervisor and computed the right generation number
- The LLM call used streaming (required for `max_tokens=32768` — another thing the spec now states explicitly)
- File parsing handled the 12 files without truncation
- The manifest was built correctly with the right hash algorithm
- The Supervisor created the branch, ran the test rig, wrote the viability report
- Gen-1 polled the Supervisor, received the result, and made the correct promote/rollback decision
- The retry prompt carried the full failure diagnostics to the next attempt

Gen-2's failures are fixable. They're exactly the kind of thing a retry prompt handles: specific test failures with stack traces, deprecated API usage, TypeErrors with clear messages. If Opus hadn't been overloaded, there would have been a Gen-3 attempt within seconds.

---

## One Architecture Clarification Worth Noting

The test rig is a verifier, not an execution environment. It starts Prime, checks `/health` and contracts, then kills it. The generation loop never runs in this path.

To actually reproduce, Gen-1 runs as a long-running process mounted against the artifacts root — not the gen-1 subdirectory, but the root, so it can write `gen-2/` in the right place for the Supervisor to find. The test rig container is ephemeral; the organism is persistent.

This seems obvious in retrospect. It wasn't obvious before we tried to run it.

---

## Where Phase 2 Actually Sits

The March 18 post put "first income" at Phase 2, estimated at $150–350 seed capital, starting after a week of Phase 1.

Revised view: Phase 1 isn't complete until self-reproduction is reliable — not just once, but consistently, across retries, with the retry prompt actually fixing the failures. Gen-2's failure was fixable. We need to see a generation loop produce a viable offspring before calling Phase 1 done.

The good news: the first obstacle (getting the loop to run at all) is cleared. The remaining work is convergence: does the retry prompt, with full failure diagnostics, guide the LLM to a passing generation? We think yes. The failures are specific and mechanical. The next run will use Sonnet for all retries, and Gen-2's `@unittest_run_loop` failures will be in the retry context.

Phase 2 — economic viability, prediction markets, paying the bills — is still the goal. We just have more respect now for how much "reliable self-reproduction" actually entails.

---

*Previous: [CAMBRIAN: What If the Spec Is the Organism?](2026-03-18-cambrian.md)*
