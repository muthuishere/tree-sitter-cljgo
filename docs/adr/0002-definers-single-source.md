# ADR 0002 — `definers.json` is the single source of truth

Status: **ACCEPTED**, implemented.

## Context

Because the grammar has no declaration node ([ADR 0001](0001-adopt-dont-fork.md)),
**every** consumer must know the same list of head symbols:

| consumer | needs it to |
|---|---|
| nvim / Helix / Zed | highlight definers as `@keyword` |
| Emacs | apply `defun` indentation |
| VS Code | scope them in TextMate |
| ctx-optimize | emit them as graph nodes |

Four hand-maintained copies. They drifted immediately: bri's `defcommand` /
`defcommands` (ADR 0078/0080) were missing from **all four**, while
`defroute` / `defroutes` were in three of them.

## Decision

`definers.json` is the source. `tools/gen-editors.js` writes the rest:

```
definers.json ──► queries/generated/definers.scm        nvim · Helix · Zed
              ──► editors/emacs/generated/definers.el    Emacs
              ──► editors/vscode/generated/definers.json VS Code (TM injection)
              ──► ctxoptimize/cljgo.json                 ctx-optimize
```

```sh
npm run gen      # write
npm run check    # CI: fail if any output is stale
```

Hand-written definer patterns were removed from `queries/cljgo-highlights.scm`,
`editors/emacs/cljgo.el` and the tmLanguage so there is exactly one owner.
Nothing under a `generated/` directory is edited by hand.

### Node, not Go

The generator is Node because this repo's toolchain is Node
(`package.json`, `tree-sitter generate`). Zero dependencies.

### VS Code uses an injection

VS Code merges an *injection grammar* into the host scope, so the generated
file is wired via a second `grammars` entry (`injectTo: source.cljgo`) rather
than rewriting the hand-maintained `cljgo.tmLanguage.json`.

## Consequences

- Adding a definer is one JSON entry plus `npm run gen`.
- A namespace-qualified head (`clojure.core/in-ns`) is emitted **only** to the
  ctx-optimize pack: the tree-sitter query matches on `!namespace` so it cannot
  match one, and `put-clojure-indent` takes a bare symbol.
- `quoted_name: true` marks forms whose name is a quoted symbol; they become a
  separate pack rule with `name_unwrap`.

## What is deliberately absent

**core.async contributes no definers.** `go`, `go-loop`, `thread`, `alt!`,
`alt!!` are *control flow* — they bind nothing at the top level — so they stay
in `queries/cljgo-highlights.scm` and out of `definers.json`. The macros
themselves are picked up normally, because `core/async.cljg` writes them as
`(defmacro go-loop …)`.

Likewise `default` / `defaults` / `default-on` appear in head position in real
cljgo code but are ordinary functions whose names merely begin with `def`.
Matching is exact-text, so they are never mistaken for definers.

`definers.json` records both, so neither is re-litigated.
