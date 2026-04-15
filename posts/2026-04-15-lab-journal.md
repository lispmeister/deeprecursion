---
title: "The Lab Journal: Writing the Laboratory Notebook, for Agents"
date: 2026-04-15
meta: "Engineering Practice"
url: https://lispmeister.github.io/deeprecursion/posts/2026-04-15-lab-journal.html
---

# The Lab Journal: Writing the Laboratory Notebook, for Agents

Forty-six dated entries. Eleven days of compressed work. One self-reproducing code factory. The artifact that held it together wasn't the git history or the issue tracker — it was a plain-text lab journal, adapted from a 1985 chemistry textbook.

We've extracted the setup into a standalone repo: [**github.com/lispmeister/lab-journal**](https://github.com/lispmeister/lab-journal). Drop it into any project, paste a block into `CLAUDE.md`, and your AI agent starts keeping a proper laboratory notebook.

Here's why that matters more than it sounds.

---

## Commit Messages Stop Scaling

Every developer knows the problem. You open a six-month-old commit. The message says `fix prime_runner bug`. You don't remember writing it. The diff is a one-line change. Why did that one line matter? What did you try first? Was there a test that caught it? Did you consider an alternative and reject it?

Git records *what changed*. It doesn't record *why you tried six things first*, *what hypothesis you were testing*, or *what the failure looked like before the fix*. For human-only work on a well-understood codebase, you can usually reconstruct the context from memory. For AI-assisted work on a novel system, you cannot.

Cambrian is a self-reproducing code generator with a 13k-token spec, an LLM-in-the-loop, a Docker-based test rig, and a supervisor managing generations. In eleven days, the project went from "bootstrap spec" to "80% viable offspring, 305 tests passing, two M3 features scaffolded." No single human could hold that arc in their head, and the AI assistant doesn't persist across sessions. A commit log showed ~200 commits with messages like `fix streaming bug` and `add manifest validation`. That log was useless for understanding the project's state.

The lab journal was the thing that made the arc legible.

---

## Kanare's Four Principles

The setup adapts Howard Kanare's *Writing the Laboratory Notebook* (American Chemical Society, 1985) — the standard reference for scientific record-keeping. Kanare's rules were written for chemists and biologists working in legal/IP contexts, but they transfer almost unchanged to software engineering with LLM agents.

*[Interactive: toggle between "commit message only" and "lab journal" views of the same real incident from the Cambrian project.]*

**Permanence.** Entries are append-only. Never delete, never rewrite. If you were wrong, you add a dated correction referencing the original — you don't erase the record. This matters because being wrong is data. The Cambrian journal has entries where the diagnosis was wrong (see journal-2026-04-03: initial diagnosis blamed the LLM; Phase 3 of the same entry records the actual root cause in `prime_runner.py`). Both halves stay. The mistake is part of what we learned.

**Immediacy.** Record as you work, not afterward from memory. Memory compresses, rationalizes, and smooths over the messy parts — the dead ends, the wrong hypotheses, the "why did I think that would work" moments. Those are exactly the parts future-you (or the agent in a new session) needs to see. An AI assistant doing a session has no memory to smooth anyway; writing as it works is the only recording that survives.

**Self-containment.** Every entry stands alone. A reader unfamiliar with the project opens any single entry and can understand what was attempted, what happened, and what was concluded. This sounds pedantic until you watch an agent pick up a project after a context compaction — the journal is sometimes the only durable state between sessions.

**Completeness.** Record what happened, not what you wish had happened. A failed experiment documented honestly saves every future session from repeating it. The six failed Cambrian generations that shared a Python 3.14 quoting bug (journal-2026-03-27) were only valuable in aggregate because each one was recorded as it failed.

One principle Kanare implies but doesn't name explicitly, we made explicit:

**Witnessing.** Every entry is signed, dated (full ISO timestamp), and linked to verifiable artifacts — commit hashes, issue IDs, spec versions. The signature and commit hash bind the entry to the record. The entry can't claim something happened that the git log contradicts.

---

## How We Used It in Cambrian

A concrete example. On 2026-04-03, a campaign ran ten generations. Zero were viable. All ten failed at the manifest stage with the same error.

The commit log says: `fix: use module form in prime_runner manifest`. One line.

The journal entry (`journal-2026-04-03.md`) records:

- The run parameters, environment variables, and spec version used
- The supervisor logs showing each generation's stage-by-stage failure
- The initial diagnosis (LLM consistently generating wrong `entry.start` — wrong, it turned out)
- The proposed fix (auto-normalize in test rig — also wrong)
- Phase 3: the *actual* root cause (`supervisor/prime_runner.py:146` hardcoding the wrong command — the manifest is not LLM-generated at all)
- The one-line fix with the before/after
- A follow-up analysis finding 82% of "LLM failures" in this period were actually infrastructure bugs
- Two new beads (cambrian-jpoq, cambrian-xmtq) for follow-up improvements
- Signed, timestamped, linked to commit 3d22a0b

Seventeen days later, when Part 14 of this blog was written, every claim in it traced back to a specific journal entry with specific evidence. I did not have to remember any of it. The journal did.

---

## The Hypothesis Table

The template has one structural element worth highlighting. Every entry that makes a testable change includes a small table:

| Change | Hypothesis (before run) | Measurement (after run) | Evidence |
|--------|-------------------------|-------------------------|----------|
| structlog lint gate | +5–10 pp viability | 8/10 viable; +12 pp vs baseline | report path |

The discipline is: fill in the hypothesis column *before* running the experiment. Record the measurement *after*. Link to the evidence.

This is the one place where software engineering typically diverges from laboratory science. We usually don't pre-register predictions. But for AI-assisted work, pre-registration is oddly valuable — it forces you (or the agent) to articulate what success would look like before the LLM starts generating code that tends to pattern-match on success language regardless of outcome. A predicted measurement of "+5–10 pp" followed by "+12 pp" is a real datum. "It worked!" is not.

---

## Why This Matters More for AI Sessions

Three things are specific to AI-assisted engineering.

**Context compaction erases short-term memory.** Long sessions with Claude or similar agents eventually compact prior conversation to stay under context limits. Anything not persisted outside the conversation is gone. The journal is the persistence layer. When `bd ready` shows the agent a list of open issues, and the journal shows the agent what was tried yesterday, session recovery takes seconds instead of rediscovery.

**The writer is the reader.** In traditional lab work, the scientist writing the notebook is also the scientist who'll reread it next week. With AI agents, the writer is often a different instance than the reader — a fresh session, a new conversation, sometimes a different model. The journal format is the interface between them. Self-containment stops being a nicety and becomes the protocol.

**The append-only constraint prevents retroactive narrative cleanup.** Agents are trained on text that presents conclusions as if they were inevitable. Left to write freely, an agent tends to narrate a session as a smooth march toward a working solution, eliding the false starts. Append-only means false starts stay visible. When the 2026-04-01 summary entry synthesized 21 prior sessions, the synthesis was honest because the substrate was honest.

---

## What's in the Repo

`lab-journal/TEMPLATE.md` — the per-session template, with goals, hypothesis table, and witness footer.

`lab-journal/index.md` — the master index, one row per entry. Keeps the table of contents navigable years later.

`CLAUDE.md` — the agent instructions. Paste into your project's `CLAUDE.md`. Covers: starting a session, working during a session, ending a session, the append-only rule, and how to update the index.

That's the whole tool. Three files and a convention. No daemon, no service, no dependency. It runs because the agent follows the rules in `CLAUDE.md`, and those rules are enforced by being plain text a human can audit.

---

## How to Adopt It

From the [lab-journal repo](https://github.com/lispmeister/lab-journal):

1. Copy `lab-journal/TEMPLATE.md` and `lab-journal/index.md` into your project's `lab-journal/` directory.
2. Paste the agent-instructions block from the repo's README into your project's `CLAUDE.md`.
3. Commit.

Next session, the agent will create a dated entry, log as it works, fill the hypothesis table before running experiments, sign off with the commit hash, and update the index. You get a durable engineering record that survives compaction, onboards fresh sessions, and captures the things commit messages can't.

The overhead is small — maybe two or three minutes per session for the footer and index update. The payback is any moment you need to know why something is the way it is, and the git log doesn't answer the question.

---

## The Honest Caveat

None of this is novel. Kanare wrote the book in 1985. Experimental scientists have kept bound, witnessed lab notebooks for centuries. The contribution here is a *translation*: taking practices built for paper notebooks in chemistry labs and rendering them as a markdown convention that AI coding agents will actually follow.

The reason it works is the same reason it worked for chemists: the discipline is minimal, the artifact is inspectable, and the cost of skipping an entry shows up later, predictably, as a forgotten decision.

Write it down. Sign it. Don't rewrite it.
