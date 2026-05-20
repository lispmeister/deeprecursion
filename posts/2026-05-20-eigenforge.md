---
title: "Eigenforge: Control Fabric for AI-Era Infrastructure"
date: 2026-05-20
meta: "Agent Architecture"
url: https://lispmeister.github.io/deeprecursion/posts/2026-05-20-eigenforge.html
---

# Eigenforge: Control Fabric for AI-Era Infrastructure

AI agents are getting good enough to participate in real operations. That does not mean they should get root access to the world.

Eigenforge is an experiment in the opposite direction: a reflective control layer where AI cognition can help operate infrastructure, but only through explicit contracts, signed authority, durable records, adapters, policy checks, and eventually quorum.

The first prototype controls one boring thing: a fan.

If CO2 rises above 1000 ppm, propose fan on. If CO2 falls below 500 ppm, propose fan off. If the sensor is stale, do nothing physical and record a signed alert.

That sounds small. It is deliberately small. The point is not the fan. The point is the shape of the authority boundary.

---

## The Motivation

Most agent demos skip the part where the agent touches the real world.

They show planning, tool calls, browser automation, code edits, maybe a dashboard. But infrastructure work has a different failure model. A bad action is not just an incorrect answer. It can move an actuator, exhaust a resource, corrupt a record, or mask a fault.

So the question is not:

> Can an AI decide what should happen?

The question is:

> Can an AI participate in a control loop without becoming the authority?

Eigenforge treats that as the core design constraint.

---

## The Shape of V1

V1 is an Elixir/OTP umbrella with five boundaries:

```text
                   Dashboard
                  read-only
                      ^
                      | projections
Simulator / HA -> Mailbox -> Core OODA -> signed command -> IO adapter -> fan
                      |        |
                      |        v
                      |   signed ledger
                      v
                  receipts
```

The core does not directly twiddle devices. IO adapters do not decide policy. The dashboard does not write. The mailbox routes and records. The ledger is append-only. Each component is intentionally less powerful than it could be.

That is the architecture in one sentence: make every boundary boring enough to audit.

---

## The Control Loop

The runtime path is a small OODA loop:

1. Observe: normalize the sensor snapshot and check freshness.
2. Orient: load signed device config and derive current room state.
3. Decide: propose an action, check policy, check capability grants.
4. Act: append signed events, issue a signed command envelope, observe the result.

In V1, the "AI" part is still a deterministic rule stub. That is intentional. The project is building the control fabric before trusting cognition inside it.

Later, an AI reasoner can replace the rule stub. The surrounding authority model should not have to change.

*[Interactive illustration in the HTML version: toggle fresh CO2, stale CO2, and already-on states to see which ledger events are appended and whether a command envelope is issued.]*

---

## What Gets Recorded

Eigenforge treats the durable record as part of the control system, not as observability garnish.

Each core node owns a local SQLite decision/action ledger. Events are signed with HMAC-SHA256, hash-chained, append-only, projected into mutable read models for the dashboard, and verified before new control work begins.

A stale CO2 reading is not just ignored. It becomes evidence:

```text
sensor snapshot
  -> reasoner outcome recorded
  -> policy decision recorded
  -> stale snapshot denied
  -> no actuator command
```

That distinction matters. "Nothing happened" is not enough. The system needs to know that nothing happened because policy denied action on stale input.

---

## Why Elixir/OTP

This kind of system wants boring concurrency: supervised processes, explicit message flow, fault isolation, restart behavior, and components that can crash without turning every failure into global ambiguity.

Elixir/OTP is a good fit because Eigenforge is not primarily a model-serving problem. It is a control-boundary problem.

The interesting work is not "call an LLM." It is what counts as authority, what must be signed, what gets persisted, what can be replayed, what survives restart, and what evidence is needed before touching the physical world.

OTP gives the runtime vocabulary for that.

---

## The Long Vision

V1 is a one-core prototype. The direction is multi-core quorum.

Instead of one local decision ledger authorizing an action, future versions can require a quorum certificate: multiple cores observe, decide, sign, compare, and only then allow IO to execute.

```text
snapshot -> Core A --\
snapshot -> Core B ----> quorum certificate -> command
snapshot -> Core C --/
```

That is the larger bet: AI-operated infrastructure should look less like an omnipotent assistant and more like a control fabric with inspectable cognition inside bounded authority.

---

## The Principle

Eigenforge starts with a fan because the smallest actuator is enough to expose the real problem.

If an agent can affect the world, then the system around it needs answers to hard questions:

Who authorized this? What policy allowed it? What input was fresh enough? What command was issued? What happened afterward? Can we prove the record was not rewritten? Can the dashboard observe without becoming authority?

Eigenforge is an attempt to make those answers concrete.

Not a smart home demo. Not an agent framework. A control fabric where intelligence can participate without being trusted blindly.

The Eigenforge repository is here: [github.com/lispmeister/eigenforge](https://github.com/lispmeister/eigenforge).
