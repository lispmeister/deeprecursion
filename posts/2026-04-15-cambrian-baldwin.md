---
title: "CAMBRIAN: The Baldwin Effect"
date: 2026-04-15
meta: "Agent Architecture"
series: "Self-Improving Agents"
series_part: 14
prev: "CAMBRIAN: It Reproduces"
prev_url: 2026-03-29-cambrian-reproduces.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-04-15-cambrian-baldwin.html
---

# CAMBRIAN: The Baldwin Effect

Part 13 ended with five promoted offspring and an open question: would evolution optimize for the wrong things? Seventeen days later, the answer is more interesting than yes or no. Selection found real quality differences between viable organisms — and the distiller that reads them is now automated.

---

## The Journey to 80%

M1 hit 50% first-try viability on a 10-generation run. We expected M2 to do better. It did not.

Gens 11–16 (six attempts) returned just one viable organism. Gens 17–19: zero. Gens 20–29: zero. Ten consecutive failures at the manifest stage with the same error: `python src/prime.py` rejected by the new validation gate.

The validation gate was working. The gate was catching the Supervisor.

**`supervisor/prime_runner.py` line 146 hardcoded `"start": "python src/prime.py"`.** The manifest isn't LLM-generated — the Supervisor builds it programmatically. Every generation since gen-16 had been failing because trusted infrastructure, not the organism, was ignoring the spec. We'd patched the spec three times (v0.14.1 → v0.14.2 → v0.14.3) and enriched the system prompt, believing the LLM was the problem. A one-line fix to a file that never read those spec updates unblocked the entire pipeline.

The lesson isn't "check the Supervisor too." It's more specific than that: **when a failure repeats across N independent LLM samples, the bug is almost never in the LLM.** Sampling noise doesn't look like N-for-N. It looks like 3-of-10. Ten-of-ten is a deterministic bug in the code path the samples all flow through.

With the prime_runner bug fixed and the system prompt enriched with concrete anti-pattern examples, the next 10-generation campaign (gens 39–48) returned 8 viable organisms.

<div class="stat-row">
  <div class="stat-cell">
    <div class="stat-label">M1 run</div>
    <div class="stat-value">50%</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">Pre-fix era</div>
    <div class="stat-value">4%</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">Confidence runs</div>
    <div class="stat-value">55%</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">Best campaign</div>
    <div class="stat-value">80%</div>
  </div>
</div>

---

## All Eight Passed. They Weren't Equal.

Eight organisms passed the same test rig, the same fitness vector, the same viability gate. Side-by-side, they weren't interchangeable. A differential code audit across the eight viable gens from the 39–48 campaign found four patterns that cleanly separated top-ranked from bottom-ranked:

- **`time.monotonic()` vs `time.time()`** for uptime measurement. System clocks jump backward on NTP adjustments. Six of eight got it wrong.
- **Exponential backoff** in the Supervisor client. One of eight implemented it. The others used fixed delays or no retry loop.
- **Specific exception types** (`aiohttp.ClientError`) vs bare `except Exception`. One of eight.
- **Structlog API correctness.** Two of eight passed `event=` as a kwarg, duplicating the positional argument.

*[Interactive: select a pattern to see which of the 8 viable organisms had it — click a generation to see its full profile.]*

The top-ranked organism (Gen-42) had three of four. The bottom-ranked (Gen-47) had zero and an anti-pattern. All eight passed. Fitness found the ordering anyway.

---

## The Baldwin Effect, Automated

In biology, the Baldwin Effect describes phenotypic traits — things organisms learn within a lifetime — eventually becoming genetic. A skill acquired through experience becomes an instinct a generation later.

The same pattern is operational here. The spec is the genome; the generated code is the phenotype. When differential analysis finds that the best-ranked phenotypes share a pattern not required by the spec, that pattern is a candidate genomic upgrade. Two such patterns (monotonic time, exponential backoff with a concrete code example) were hand-encoded into CAMBRIAN-SPEC-005 v0.14.4 after the 39–48 campaign. The next campaign should produce more 8-of-8 quality outputs, because the "learned" behavior is now in the spec every generation reads.

Hand-encoding is the part that didn't want to stay hand-encoded. `scripts/distill_campaign.py` is an AST-based post-campaign analyzer: parse every Python file in every viable gen, walk the trees for known patterns (positive and negative), rank gens by a composite fitness score, and emit proposed spec amendments with code examples lifted from the best gens. Validated against the 39–48 campaign, it independently rediscovered all four patterns the manual audit had found, with the top-vs-bottom ranking signal noticeably cleaner than the viable-vs-failed signal.

The loop is closed: phenotype teaches genome, mechanically. The next step — feeding the distiller's proposals directly into `spec_mutator.py` as informed mutations — replaces random perturbation with targeted improvement in the BO loop.

---

## What This Costs

The 39–48 campaign ran on claude-sonnet-4-6 throughout. A focused comparison on a single generation showed Opus at roughly 5× the cost per attempt with no clear quality advantage for spec compliance — Opus gen-35 required a parse repair before succeeding; Sonnet gen-32 did not. The economic default is Sonnet with Opus reserved for explicit escalation.

<div class="stat-row">
  <div class="stat-cell">
    <div class="stat-label">Sonnet / gen (fresh)</div>
    <div class="stat-value">~$0.43</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">Sonnet / gen (feedback)</div>
    <div class="stat-value">~$0.47</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">Opus / gen</div>
    <div class="stat-value">~$3.83</div>
  </div>
  <div class="stat-cell">
    <div class="stat-label">10-gen campaign</div>
    <div class="stat-value">~$4.70</div>
  </div>
</div>

A feedback-enabled campaign adds about 1k input tokens per generation cumulatively, because the prior failure's diagnostics and source are threaded into the next prompt. That's the cost of Lamarckian retry: ~4% more input for the full failure context.

---

## Adversarial Review, Two Rounds

Evolution rewards whatever passes the gate. If the gate is weak, the organism learns to pass the gate.

Two rounds of adversarial review tightened the trust boundary in April. Round one stripped the Anthropic API key from the Test Rig (the Rig should never need it — only the Supervisor does), set `NetworkMode=none` on Test Rig containers so no organism can phone home, and removed dead code paths (`screen_mutation()`) that weren't wired into the promotion pipeline. Round two added reverse-run workspace isolation, sandbox controls (`no-new-privileges`, Linux capability drops), BO resume deduplication, and append-only semantics on campaign records. The specs bumped from v0.14.4 / v0.9.1 to v0.18.0 / v0.14.0 in a week.

The more interesting hardening is on the evaluation side. The Test Rig now captures `container.log` before rollback on every path — normal exit, timeout, process group kill — so failure diagnostics persist even when the container is gone. A regression-guard mechanism (M3 scaffolding) runs old baseline artifacts against the current Test Rig whenever a new generation is promoted, producing a 2×2 matrix of "new artifact vs old contracts" and "old artifact vs new tests." If an organism passes the current gate but would fail against the battery of contracts established by its ancestors, that's visible in a separate fitness dimension rather than hidden behind a binary viable/not-viable.

Binary viability is a weak gate. Binary viability against every historical contract is a much harder one to game.

---

## What We Actually Learned

Three things became clear between Part 13 and this one.

**Most "LLM failures" are infrastructure failures.** Of 29 early M2 generation attempts, 82% failed for reasons that had nothing to do with the organism's code. Stale Docker images, wrong `/venv` ownership, a hardcoded command in the Supervisor, a manifest field built by trusted code. Each looks like the LLM generating bad output. Each was actually our own bug in a layer the LLM never saw. Ten failures in a row are a deterministic bug. Always.

**Fitness ordering works before fitness pressure does.** We haven't run a selection campaign — none of the gens 39–48 were chosen over each other yet. But we can already see which ones would win. The differential phenotypic patterns are visible in a single campaign's worth of viable organisms. Selection pressure is just the mechanism that compounds what's already measurable.

**Spec precision is the rate-limiting step.** Every quality jump in the project has traced back to a spec patch, not a code patch. The streaming requirement, the aiohttp test pattern, the module-form entry command, the `time.monotonic()` rule, the backoff example. The LLM is a very literal compiler of the spec into code. Ambiguity in the spec becomes ambiguity in the code. The distiller automates the discovery of what the spec was missing.

---

## What Comes Next

The infrastructure for M2 Stage 1 is built, tested (305 tests passing), and rubric-backed (`summarize_m2_results.py` reports baseline-vs-mutation deltas against a checklist). What's outstanding is an actual spec-mutation campaign where the BO loop proposes spec changes, the mini-campaigns evaluate them, and the distiller-informed mutator feeds phenotypic wins back as targeted spec amendments rather than random perturbations.

Beyond that, M3's counterfactual baseline battery — already scaffolded — will run every promoted generation against a growing library of contracts extracted from every prior generation. A fitter organism will be one that satisfies not just the current spec but the intersection of every spec its lineage has passed. That's a much harder surface to game than a single-campaign fitness vector.

The honest status: reproduction works reliably. Ordered fitness across viable organisms is measurable and automatable. Selection pressure itself — actually choosing between spec variants and letting the winners propagate — is the next thing to run and the next thing that can surprise us.

---

*Previous: [CAMBRIAN: It Reproduces](2026-03-29-cambrian-reproduces.md)*
