# ADR 0003 — Language add: the `decl_rules` contract

Status: **ACCEPTED**, implemented. `decl_rules` is merged into ctx-optimize
`main` and ships from 0.10.0; the pack's location was corrected by
[ADR 0004](0004-pack-ships-where-it-is-read.md).

How `ctx-optimize languages add` works for cljgo, and why it needs a pack
format that did not previously exist.

## The problem

ctx-optimize's pack format maps **AST node type → node kind**:

```json
"decls": { "function_declaration": "function", "class_declaration": "class" }
```

That assumes a declaration *has* a node type. **Every Lisp violates this.**
In cljgo there is no `defn` node — `(defn fetch-user [] …)` is a `list_lit`
whose *head symbol* carries the meaning and whose *second element* is the name.

So `ctx-optimize languages add <this repo>` produced `decls: {}`, and `add`
then hard-rejected it:

```
name, exts and decls are required
```

The pack was unbuildable **by construction**, not by misconfiguration.

### The trap: mapping `list_lit → function`

The obvious rescue is to map the only container node there is, taking the name
from the head. On an 11-line file that produced:

```
function L4-L4   | def          ← should be `config`
function L6-L8   | defn         ← should be `fetch-user`
function L8-L8   | defn.get     ← a CALL SITE, not a declaration
function L7-L7   | defn.println ← a CALL SITE
function L1-L2   | ns
                               ← `handler` never appears at all
```

Two nodes named `defn`, call sites emitted as declarations, and
`query "fetch-user"` returning nothing. **Confidently wrong** — worse than no
support, and the one outcome a code graph must never produce.

## Decision

Match on the **head symbol**; read the name from the **next element**.

```json
{
  "node": "list_lit",
  "head_type": "sym_lit",
  "name_type": "sym_lit",
  "skip_inside": ["quoting_lit", "syn_quoting_lit", "dis_expr"],
  "head_match": { "defn": "function", "def": "variable", "ns": "module" }
}
```

| field | meaning |
|---|---|
| `node` | container node type to test |
| `head_type` | required type of the **first** named child |
| `head_match` | that child's literal text → the kind emitted |
| `name_type` | required type of the **second** named child (the name) |
| `name_unwrap` | wrapper types to step over first |
| `skip_inside` | ancestor types that void a match |

A pack may use `decls`, `decl_rules`, or both.

### Two rules, not one

`(clojure.core/in-ns 'bri.cli)` — how much of cljgo's core declares its
namespace — misses twice over: the head is **namespace-qualified**, and the
name is a **quoted symbol** (`quoting_lit`, not `sym_lit`). Hence a second rule
with `name_unwrap` and the qualified head listed literally in `head_match`.

This was not theoretical: 34 files declared their namespace this way and were
entirely invisible. Adding the rule took module nodes from **8 to 39**.

## Why it is honest

The governing rule is *emit a fact only if it is read literally out of the
file*. Every failure mode here lands on the **under-claim** side:

| case | behaviour |
|---|---|
| `s/def`, `:rename` aliases | head text no longer matches exactly → nothing emitted |
| `(defsomething x)` from an app's own macro | not in `head_match` → nothing emitted until added |
| `(defn ^:private f …)` | metadata in the name slot → form skipped, not named wrongly |
| `` `(defn ~name …) `` in a macro body | code being *constructed*, not defined → excluded by `skip_inside` |

`skip_inside` is a **structural** test against real grammar nodes
(`quoting_lit`, `syn_quoting_lit`), not a heuristic.

### Measured on cljgo's own source (658 files)

| | |
|---|---|
| definer-headed forms | 1,372 |
| resolved to a literal name in second position | 1,362 (99.3%) |
| inside a quote / syntax-quote | 5 (0.36%) — excluded |
| **wrong facts emitted** | **0** |

The 7 nodes that *are* named after a macro are correct: they are
`(defmacro defcommand …)` and friends — the definitions of the macros
themselves, including cljgo's own `defn` at `core/core.clj:30`.

## Why data, not grammar

A cljgo program **creates defining macros at run time**. `core/bri/cli.cljg`
defines `defcommand`; an app defines `defjob`. No parser can know them — a
table can, and `LoadPacks` reads `<repo>/.ctxoptimize/grammars/` **before** the
machine-wide directory, so a project's definers ship with the project. No
release, no registry entry, no adapter.

This is also the reason [ADR 0001](0001-adopt-dont-fork.md) refuses to put
`defn` in the grammar: the same head-symbol heuristic baked into a parser is a
regeneration and a fork divergence; as data it is a one-line edit.

## Defects fixed alongside

1. **`grammar build` reported success for a pack that cannot load** — it
   printed *"pack ready … next `ctx-optimize add` picks it up"* for an
   empty-`decls` pack that `add` rejects one command later. Now fails at build
   time, naming the homoiconic case.
2. **Extensions were guessed from the repo name** — `.cljgo` from
   `tree-sitter-cljgo`, `.clojure` from `tree-sitter-clojure`: an extension
   nobody uses, silently matching nothing while looking configured.

## Verification

```sh
npm run ctx-pack        # wasm beside the committed .ctxoptimize/grammars/cljgo.json
ctx-optimize up && ctx-optimize query "fetch-user"
```

Measured against cljgo's `core/`: 868 nodes, 33 modules, real names throughout.

## Not claimed

Measured on cljgo/Clojure **only**. Fennel, Janet, Elisp and Racket are the
same shape and should work, but no corpus has been run — do not advertise them
until one has.
