# tree-sitter-cljgo

Language support for **[cljgo](https://github.com/muthuishere/cljgo)** — Clojure
hosted on Go. A [Tree-sitter](https://tree-sitter.github.io/) grammar, editor
queries for every major editor, and a [ctx-optimize](https://github.com/muthuishere/ctx-optimize)
pack, all driven from one file.

```
definers.json ──► queries/generated/definers.scm       nvim · Helix · Zed
              ──► editors/emacs/generated/definers.el   Emacs
              ──► editors/vscode/generated/definers.json VS Code
              ──► .ctxoptimize/grammars/cljgo.json      ctx-optimize
```

## Why one file

cljgo is **homoiconic**: a definition has no node type of its own.
`(defn fetch-user [] …)` is an ordinary `list_lit` whose *head symbol* (`defn`)
carries the meaning and whose *second element* (`fetch-user`) is the name.

So every tool that wants to find a definition needs the same list of head
symbols — editors to highlight and indent them, ctx-optimize to emit them as
graph nodes. Four hand-kept copies drift. `definers.json` is the source.

## Add a defining form

Your app defines `(defjob nightly-sync …)`? One entry:

```json
{ "head": "defjob", "kind": "function", "indent": "defun" }
```

`kind` is the ctx-optimize node kind (`function`, `variable`, `macro`, `class`,
`interface`, `module`, `test`); `indent` is the clojure-mode indent spec.
Then:

```sh
npm run gen        # rewrite all four outputs
npm run check      # CI: fail if any output is stale
```

Never edit anything under a `generated/` directory.

## File extensions

cljgo loads four (`pkg/eval/libload.go`, ADR 0055/0068): `.cljgo` `.cljg`
`.clj` `.cljc`. `.clj` / `.cljc` already map to Clojure in every editor, so in
practice only `.cljgo` and `.cljg` need adding.

## Editors

### Neovim

```sh
mkdir -p ~/.config/nvim/after/queries/clojure
for q in cljgo-highlights generated/definers; do
  { echo ';; extends'; cat queries/$q.scm; } \
    >> ~/.config/nvim/after/queries/clojure/highlights.scm
done
for q in injections locals; do
  { echo ';; extends'; cat queries/$q.scm; } \
    > ~/.config/nvim/after/queries/clojure/$q.scm
done
```

```lua
vim.filetype.add({ extension = { cljg = "clojure", cljgo = "clojure" } })
```

### Emacs

```sh
mkdir -p ~/.emacs.d/lisp
cp editors/emacs/cljgo.el editors/emacs/generated/definers.el ~/.emacs.d/lisp/
```

```elisp
(add-to-list 'load-path "~/.emacs.d/lisp")
(require 'cljgo)
(require 'definers)
(cljgo--apply-definer-indents)
(add-to-list 'auto-mode-alist '("\\.cljgo?\\'" . clojure-mode))
```

### VS Code

VS Code uses TextMate scopes, not tree-sitter. The generated file is wired as
an injection in `editors/vscode/package.json`:

```sh
cd editors/vscode && npx @vscode/vsce package && code --install-extension cljgo-0.0.1.vsix
```

Or just map the file types in `settings.json`:

```json
{ "files.associations": { "*.cljg": "clojure", "*.cljgo": "clojure" } }
```

See [`editors/REGISTRATION.md`](editors/REGISTRATION.md) for why `.clj` /
`.cljc` are deliberately not claimed.

### Helix

```toml
[[language]]
name = "clojure"
file-types = ["clj", "cljs", "cljc", "cljg", "cljgo", "edn", "bb"]
```

Helix replaces rather than extends queries — copy the stock clojure queries
into `~/.config/helix/runtime/queries/clojure/` and append this repo's, cljgo
patterns first. In `locals.scm`, rename `@local.definition.var` →
`@local.definition`.

### Zed

```json
{ "file_types": { "Clojure": ["cljg", "cljgo"] } }
```

## ctx-optimize

The pack is committed at `.ctxoptimize/grammars/cljgo.json` — the path
ctx-optimize reads, and reads *before* the machine-wide directory. Only the
parser wasm is missing from a fresh clone, because `*.wasm` is gitignored:

```sh
npm run ctx-pack        # builds .ctxoptimize/grammars/cljgo.wasm beside the JSON
ctx-optimize up
ctx-optimize query "fetch-user"
```

`ctx-pack` builds from this repo's own `grammar.js` — no network — and keeps the
committed mapping rather than overwriting it. Verified against an **empty**
machine-wide grammars dir: 20 function, 22 variable and 7 module nodes across the
7 `.cljg` examples, none named after a defining macro. See
[ADR 0004](docs/adr/0004-pack-ships-where-it-is-read.md).

To use cljgo in **your** project, copy that JSON into your repo's own
`.ctxoptimize/grammars/` and add your project's definers to `head_match`; the
repo-local pack wins over the machine-wide one, so your definers travel with your
code.

### Why the pack uses `decl_rules`

ctx-optimize's normal pack format maps **node type → kind**
(`function_declaration` → `function`). cljgo has no such node type. Mapping
`list_lit → function` produces garbage — every node named after the macro:

```
function  defn      ← should be fetch-user
variable  def       ← should be config
                    ← `handler` never appears at all
```

`decl_rules` matches on the head symbol and reads the name from the next
element. Two rules, because `(clojure.core/in-ns 'bri.cli)` — how much of
cljgo's own core declares its namespace — wraps the name in a quote.

It is **literal and under-claims**: `s/def` and `:rename` aliases miss,
`(defn ^:private f …)` is skipped rather than named wrongly, and a `defn`
inside a syntax-quote is a macro *constructing* code, not defining it, so it's
excluded structurally. Measured on cljgo's own source: 1,372 definer-headed
forms, 1,362 literal names, **0 wrong**.

> **Requires ctx-optimize with `decl_rules` support.** Older versions reject
> the pack with `name, exts and decls are required` — a pack keyed on node
> types cannot express a homoiconic declaration.

## The grammar

Identical to Clojure's syntax, derived from
[`tree-sitter-clojure`](https://github.com/sogaiu/tree-sitter-clojure) and
generated as `cljgo`. This is permanent, not provisional: cljgo's rule is that
**nothing it adds may shadow or change `clojure.core` semantics**, so every
cljgo addition (`require-go`, the Result/Option track, core.async on Go
channels, bri's `defroute` / `defcommand`) is an ordinary macro, not new
syntax. Node types match upstream 35/35.

That is also why definers are **data, not grammar**: a cljgo program creates
defining macros at run time (`core/bri/cli.cljg` defines `defcommand`), so no
parser can know them — but a table can.

```sh
tree-sitter generate     # after editing grammar.js
tree-sitter build
```

## Verifying a change

```sh
npm run check                                  # generated outputs in sync
tree-sitter build
for f in examples/*.cljg; do                   # zero ERROR nodes
  tree-sitter parse "$f" | grep -q ERROR && echo "ERROR: $f"
done
tree-sitter query --captures queries/generated/definers.scm examples/cli.cljg
```

A query that *loads* is not a query that *matches* — when you add a definer,
extend an `examples/*.cljg` to exercise it and confirm it appears in the
`--captures` output.

## Design decisions

| # | |
|---|---|
| [0001](docs/adr/0001-adopt-dont-fork.md) | The grammar stays stock Clojure syntax — permanent, not provisional |
| [0002](docs/adr/0002-definers-single-source.md) | `definers.json` is the single source of truth |
| [0003](docs/adr/0003-language-add-decl-rules.md) | Language add: the `decl_rules` contract |

## Lineage

Forked from `sogaiu/tree-sitter-clojure` (same license). Credit to its authors,
and to its `doc/scope.md`, which reasoned through why `defn` cannot live in a
Clojure grammar — and measured the error rate to prove it. This repo follows
that conclusion: the grammar stays primitives-only, and definitions are found
by the query and pack layers instead.
