---
title: "What if the LLM Thought in S-Expressions?"
date: 2026-02-13
meta: "Responding to Can Bölük's 'The Harness Problem'"
url: https://lispmeister.github.io/deeprecursion/posts/2026-02-13-sexp-native-editing.html
---

# What if the LLM *Thought* in S-Expressions?

*February 13, 2026 · Responding to [Can Bölük's "The Harness Problem"](https://blog.can.ac/2026/02/12/the-harness-problem/)*

## The Thesis

Bölük's hashline insight is that **the model doesn't need to reproduce content to prove it knows what it's editing** — a 2-char hash is enough. Brilliant. But it still operates in line-space. Code isn't lines. Code is a tree. What if we went further?

In Lisp, **the syntax *is* the AST**. There is no lossy parse step, no formatting to preserve, no ambiguity between the concrete and abstract tree. The parentheses aren't syntax sugar — they're structure made visible. This means an LLM that "reads" Lisp is *already reading a serialized AST*. It just doesn't know it.

So the question becomes: what if we made it know?

### The key realization

Hashline gives each *line* an identity. But lines are arbitrary. A function might span 1 line or 40. What we really want is to give each **semantic node** an identity. In Lisp, every parenthesized form is a node. So we can do something no other language allows cleanly:

**What the programmer writes:**

```lisp
(defn factorial [n]
  (if (<= n 1)
    1
    (* n (factorial (dec n)))))
```

**What the model sees:**

```lisp
§a(defn factorial [n]
  §b(if §c(<= n 1)
    1
    §d(* n §e(factorial §f(dec n)))))
```

Each `§`-prefixed tag is a **content-addressed node ID** — a short hash of the s-expression rooted at that paren. Not a line number. Not a character offset. A *semantic address*. The model doesn't point at "line 3" — it points at `§c`, which *means* `(<= n 1)` regardless of how the code is formatted or where it appears in the file.

> **The difference:**
> Hashline: "replace line 2, hash f1" → fragile to reformatting, insertion, reordering.
> S-expr node IDs: "replace §c" → stable across all formatting, movement, refactoring. The identity travels with the semantics.

## The Protocol

### Edit primitives

The model gets exactly four operations. No diffs. No string matching. No patches. Each operation is itself an s-expression — the model never leaves its native syntax:

```lisp
;; Replace a node's content entirely
(replace! §c (zero? n))

;; Wrap a node inside a new form
(wrap! §d (when (pos? n) ◊))  ; ◊ = the original node

;; Insert a sibling before/after a node
(insert-after! §c (println "base case hit"))

;; Delete a node
(delete! §e)
```

Notice what's *absent*. No line numbers. No character ranges. No reproduction of the old content. No whitespace. The model says *what* to change and *what to change it to*. The harness handles *where* and *how*.

### Why this is structurally superior to hashline

- **Anchor granularity:** Node (vs. line for hashline, string for str_replace)
- **Content reproduction:** Zero — model never re-states old code
- **Stale edit detection:** Exact — hash is the node, not a line approximation
- **Reorder-safe:** Yes — moving code doesn't invalidate IDs

But here's the deeper insight. **Composability.** Because each edit is itself an s-expression, edits compose naturally. You can nest them, sequence them, even write macros over them:

```lisp
(do
  (replace! §c (zero? n))
  (wrap! §a
    (defn factorial [n]
      (assert (nat-int? n) "n must be natural")
      ◊)))

;; This atomically: fixes the base case AND wraps the body in a guard.
;; The harness validates both node IDs before applying either change.
```

### The radical extension: edits as data

Because edits are s-expressions, they're **first-class data**. The harness can quote them, store them, invert them, diff them, and even ask the model to *review its own pending edits as code*. A refactoring plan isn't a blob of text — it's a program that transforms programs.

```lisp
;; Harness: "Here is your proposed edit. Confirm or revise."
(let [edit (replace! §c (zero? n))]
  (preview edit)        ;; → shows before/after in context
  (inverse edit)        ;; → (replace! §c' (<= n 1))
  (conflicts? edit)     ;; → nil
  (apply! edit))        ;; → commit
```

## Verdict

### So would this beat hashline?

**Under idealized conditions — single homoiconic language, no formatting concerns — yes, unambiguously.**

Hashline solves the *identification* problem but still operates in line-space, which is a **lossy projection** of semantic structure. A single logical change might span multiple lines or only part of a line. Lines shift when you add code above. A reformatter invalidates every hash in the file.

S-expression node addressing solves identification at the *correct* level of abstraction: the semantic unit. A node ID refers to a specific subexpression regardless of where it is in the file, how it's indented, or what's around it. The identity is *intrinsic* to the content, not to its position.

### The deeper argument: cognitive alignment

When an LLM reads Lisp, every token is *already* a tree operation: an open paren pushes a frame, a close paren pops one. The model's internal representation of Lisp code likely resembles an AST more than its representation of Python or JavaScript, because **the syntax forces tree-shaped attention patterns**.

By making the edit language also be s-expressions, you eliminate every representation conversion. Code is trees. Edits are trees. The model's cognition is trees. **Zero impedance mismatch** at any layer.

### The honest caveat

This is a **beautiful impossibility as a general solution**. You can't rewrite the world in Lisp. Bölük's hashline wins precisely because it works on *any text file in any language* with zero setup.

The interesting middle ground: use tree-sitter to give *any* language pseudo-homoiconic node IDs. You'd get most of the benefits — semantic anchoring, reorder stability, no content reproduction — without requiring Lisp. The formatting round-trip problem remains, but only for the *inserted* code, not for the targeting mechanism.

> **The real takeaway:**
> The closer the edit interface matches the model's internal representation of code, the fewer mechanical failures you get. Hashline moves from strings to lines. S-expr addressing moves from lines to semantic nodes. The ideal — which tree-sitter approximates for all languages — is editing at the *meaning* level, with the harness handling everything below.
