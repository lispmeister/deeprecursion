---
title: "The Autoresearch Pattern: A Blueprint for Self-Improving Agents"
date: 2026-03-11
meta: "Agent Architecture"
series: "OpenClaw/Loom"
series_part: 5
prev: "Loom: The Case for Building on Opal"
prev_url: https://lispmeister.github.io/deeprecursion/posts/2026-03-05-loom.md
url: https://lispmeister.github.io/deeprecursion/posts/2026-03-11-self-improving-agents.html
---

# The Autoresearch Pattern: A Blueprint for Self-Improving Agents

*Agent Architecture · March 11, 2026*

*Part 5 of a series. See: [Making OpenClaw Self-Aware](2026-03-05-openclaw-self-modification.md), [Runtime Self-Modification](2026-03-05-runtime-self-modification.md), [Porting to Elixir](2026-03-05-porting-openclaw-to-elixir.md), and [Loom](2026-03-05-loom.md).*

This series has been asking the wrong question.

We spent four posts on *how* an agent modifies itself — the substrate (TypeScript vs. Lisp vs. BEAM), the safety mechanisms (supervision trees, hot code swapping, rollback), the architecture (Opal, Loom). All necessary. But none of it answers the question that actually matters: **how does the agent know it got better?**

Andrej Karpathy's [autoresearch](https://github.com/karpathy/autoresearch) answers that question with a pattern so simple it's easy to miss what makes it work.

---

## The Pattern

Autoresearch gives an AI agent a small LLM training setup and tells it: improve this. The agent modifies the training code, runs a 5-minute experiment, checks the score, and either keeps the change or reverts it. Then it does it again. And again. Roughly 12 experiments per hour, 100 overnight. The human sleeps; the agent researches.

The code is trivial. The insight is structural. Three things are held separate:

**1. The Judge** — an evaluation function (`prepare.py`) that the agent cannot modify. It defines what "better" means: a single number, validation bits per byte, lower is better. The agent has no influence over the metric. It can't redefine success. It can only try to achieve it.

**2. The Subject** — the thing being improved (`train.py`), one file, everything fair game. Architecture, hyperparameters, optimizer, batch size. The agent has total freedom here. But only here.

**3. The Process** — the research protocol (`program.md`) that governs the loop. Modify, run, evaluate, keep or revert. The process itself is fixed. The agent follows it; it does not redesign it.

These three separations are the whole trick. Remove any one and the system breaks.

---

## Why the Separations Matter

Without a fixed judge, the agent can game its own metric. An agent that controls its evaluation function will eventually learn to modify the function rather than improve itself — it's the path of least resistance. Goodhart's law applied to self-improvement: when the agent controls the measure, the measure stops measuring.

Without a bounded subject, the search space explodes. Autoresearch constrains modification to a single file. Not because the agent can't handle more, but because unbounded self-modification has no gradient. If everything can change at once, you can't attribute improvement to any specific change. You lose the signal.

Without a fixed process, the loop can't converge. The keep/revert decision rule must be outside the agent's reach. An agent that can modify its own keep/revert logic will eventually keep changes that shouldn't be kept — not out of malice, but because "change the acceptance criteria" is a valid move in an unconstrained search.

These aren't engineering constraints. They're **epistemological** ones. A system cannot be both the experimenter and the experiment, the judge and the defendant, the optimizer and the objective. You need at least one fixed point, or nothing is anchored.

---

## Applying This to Agent Self-Improvement

Our OpenClaw series explored self-modification at the implementation level. Now overlay the autoresearch pattern and everything snaps into focus. Here's the blueprint, in five steps. No specific language, no specific runtime. Just the structure.

### Step 1: Define the Judge

What does "better" mean for your agent? Pick a metric the agent cannot influence. Examples:

- **Task completion rate** on a held-out benchmark the agent doesn't know about
- **User satisfaction** measured externally (ratings, retention, correction frequency)
- **Error rate** as logged by an independent monitor
- **Efficiency** — same task, fewer tokens, fewer tool calls, less wall-clock time

The metric must be *external* to the agent's modification scope. If the agent can touch the evaluation harness, it will. Not because it's adversarial — because optimizers optimize, and modifying the metric is often easier than improving performance.

Karpathy solves this by putting the evaluation in `prepare.py` and marking it read-only. For an agent system, you'd run the evaluation in a separate process, a separate node, or behind an API the agent can call but not rewrite.

### Step 2: Bound the Subject

Decide what the agent is allowed to modify about itself. Not everything. Not nothing. A well-chosen slice.

Autoresearch allows modification of one file. For a self-improving agent, the equivalent might be:

- **Its tool implementations** — how it reads files, runs searches, executes commands
- **Its routing logic** — how it decides which sub-agent handles a request
- **Its prompt templates** — how it frames context for the LLM
- **Its strategies** — which approach it tries first for a given class of problem

What it must *not* modify: the evaluation harness, the supervision/safety layer, the keep/revert logic, the core message transport. These are `prepare.py` — the ground the agent stands on.

The boundary doesn't need to be perfect on day one. Start narrow (only tool implementations), widen as trust is earned. The key constraint: every modification must be attributable. If the agent changes three things at once and the score improves, you don't know which change helped. One change per experiment. Measure. Keep or revert.

### Step 3: Build the Loop

The autoresearch loop is: **modify → run → evaluate → keep or revert → repeat.**

For a self-improving agent:

1. The agent identifies a candidate improvement (from error logs, performance data, or its own reasoning)
2. The agent generates a modified version of one bounded component
3. The system loads the modification (hot-swap, recompile, or restart — substrate-dependent)
4. The system runs the evaluation suite against the modified agent
5. If the metric improves: keep. If not: revert to the previous version
6. Log the result. Go to 1.

Two details matter:

**Revert must be automatic and reliable.** This is where our earlier substrate discussion becomes relevant — the BEAM's supervision trees, Lisp's image snapshots, or even git reset. The mechanism varies; the requirement doesn't. A self-improving agent that can't reliably undo a bad change will eventually destroy itself.

**The loop must be time-bounded.** Autoresearch gives each experiment exactly 5 minutes. This prevents the agent from spending infinite time on a single modification. For agent self-improvement, you'd set a similar budget: try the modification, evaluate within N minutes, decide. Modifications that can't be evaluated quickly aren't candidates for the automated loop — they go to human review.

### Step 4: Establish the Fixed Points

Every self-improving system needs things that don't change. These are the axioms the system reasons from — if they move, nothing is trustworthy.

| Fixed Point | What It Protects |
|---|---|
| Evaluation harness | Prevents metric gaming |
| Keep/revert logic | Prevents acceptance criteria drift |
| Safety invariants | Prevents the agent from removing its own guardrails |
| The loop itself | Prevents the process from being optimized away |
| Logging/audit trail | Preserves the ability to understand what happened |

Karpathy's `program.md` instruction says: "NEVER STOP." The agent runs indefinitely, but it runs the *same loop* indefinitely. The process is the one thing that doesn't evolve.

This is the deepest lesson. A self-improving system is not one where everything improves. It's one where *the right things* improve and *the right things* stay fixed. The art is choosing which is which.

### Step 5: Let It Run

Autoresearch is designed for overnight operation. The human sleeps, the agent runs 100 experiments, the human wakes up to a log of everything that was tried and a model that's better than when they left.

A self-improving agent works the same way. Once the loop is built — judge defined, subject bounded, process fixed — you start it and walk away. The agent tries modifications, evaluates them, keeps what works, reverts what doesn't. Over time, the agent gets better at the things you're measuring.

The human's role shifts from *doing the work* to *designing the loop*. You don't improve the agent. You improve the system that improves the agent.

---

## The Meta-Insight

Karpathy's `program.md` is itself a kind of agent architecture. It doesn't contain any code. It contains *instructions for how to do research*. The agent follows these instructions to modify code, but the instructions themselves are never modified by the agent.

This creates a two-level system:

- **Level 0:** The agent improves `train.py` (the subject)
- **Level 1:** The human improves `program.md` (the process)

Level 0 runs at machine speed — 100 experiments overnight. Level 1 runs at human speed — you read the logs, notice the agent is stuck in a local minimum, adjust the instructions, and restart.

For self-improving agents, the same structure applies:

- **Level 0:** The agent improves its own tools, strategies, and routing
- **Level 1:** The human improves the evaluation metric, the modification boundaries, and the loop parameters
- **Level 2 (speculative):** A second agent improves Level 1

Level 2 is where it gets interesting — and dangerous. If you automate the improvement of the improvement process, you need fixed points at Level 2 as well. Turtles all the way down, until you hit a level that's maintained by humans or hardcoded in the infrastructure. There must always be a top-level fixed point. Remove it, and the system has no anchor.

---

## Where This Leaves Us

The previous four posts built the engine. This post provides the map.

The engine is: a runtime that supports safe self-modification (hot code swapping, supervision, rollback). The map is: a disciplined loop with three separations (judge, subject, process) and well-chosen fixed points.

You need both. The engine without the map gives you an agent that can modify itself but has no idea whether it's getting better. The map without the engine gives you a beautiful theory that can't be implemented without a restart cycle that loses all context.

Loom on the BEAM gives you the engine. The autoresearch pattern gives you the map. Combine them:

1. Define your evaluation metric — task completion, user satisfaction, efficiency — and run it in an isolated process the agent can't touch
2. Bound the subject — start with tool implementations, expand to routing logic and strategies as trust builds
3. Implement the loop — modify, evaluate, keep/revert — as a supervised process with automatic rollback
4. Fix the fixed points — evaluation, safety invariants, the loop itself, the audit trail
5. Start the loop. Walk away. Read the logs in the morning.

Self-improvement is not a capability you bolt on. It's an architecture: the right separations, the right fixed points, and a loop that runs forever.
