# ADR 0001 — The grammar stays stock Clojure syntax

Status: **ACCEPTED** — permanent, not provisional.
Mirrors cljgo ADR 0017 §2.

## Decision

cljgo's grammar is `sogaiu/tree-sitter-clojure`, generated under the name
`cljgo`. It gains no rules of its own. Ever.

## Why this is permanent

cljgo's governing rule is that **nothing it adds may shadow or change
`clojure.core` semantics** — Clojure is first-class. Every cljgo addition is
therefore an ordinary macro, not new syntax:

| addition | form | new syntax? |
|---|---|---|
| Go interop (ADR 0010) | `(require-go '[strings])` | no — a list |
| Result/Option (ADR 0014) | `(ok v)`, `(let? […] …)` | no |
| core.async on Go channels (ADR 0040) | `(go …)`, `(<! ch)` | no |
| bri routing (ADR 0069) | `(defroutes api …)` | no — a macro |
| bri CLI (ADR 0078/0080) | `(defcommand add …)` | no — a macro |

A node-type diff against upstream found **zero difference in either
direction — all 35 identical**. There is nothing to fork.

## Rejected: adding `defn` and friends to the grammar

Tempting, because a `defn_form` node with `field('name', …)` would make every
downstream tool trivial. Rejected because a grammar cannot decide three things
that are resolved at macroexpansion:

1. **User-defined definers.** `defcommand` is defined *in cljgo source*
   (`core/bri/cli.cljg`). Apps define their own. A fixed grammar list is
   permanently incomplete.
2. **`defn` as data.** `(list 'defn name args body)` inside a macro is a
   symbol, not a definition.
3. **Aliasing.** `s/def`, `(:refer-clojure :rename {defn my-defn})`.

Upstream measured this: earlier attempts to add `def` and friends produced
"unacceptably high error rates" against real Clojars code
(`doc/scope.md`). Each added rule also raises parse error rates and forfeits
clean rebasing onto upstream.

## Consequence

Definitions are found by the **query and pack layers** instead — see
[ADR 0002](0002-definers-single-source.md) and
[ADR 0003](0003-language-add-decl-rules.md). Those layers are data, which is
exactly what a run-time-extensible macro system requires.
