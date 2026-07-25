# ADR 0004 — the pack ships where ctx-optimize reads it

Status: **ACCEPTED**, implemented.

Where the generated ctx-optimize pack lives, and why a fresh clone of this repo
produced no declarations at all.

## The problem

`tools/gen-editors.js` wrote the pack to **`ctxoptimize/cljgo.json`**. That path
is not read by anything. `LoadPacks` searches exactly two directories:

1. `$CTX_OPTIMIZE_GRAMMARS` (default `~/ctxoptimize/grammars/`) — machine-wide
2. `<repo>/.ctxoptimize/grammars/` — travels with the repo, read first

`ctxoptimize/` is neither: no leading dot, no `grammars/` segment. It was a
directory this repo's generator invented for its own output.

Measured on a clone of this repo with an empty machine-wide grammars dir:

```
$ CTX_OPTIMIZE_GRAMMARS=<empty> ctx-optimize add cljgo --path .
code: 60 nodes, 71 edges
nodes from .cljg/.cljgo: NONE
```

Sixty code nodes — all from the grammar's own `src/*.c` and `tools/*.js`. **Zero
declarations from the 7 `.cljg` example files**, the language this repo exists to
support.

This was masked on the author's machine, where `~/ctxoptimize/grammars/cljgo.*`
had been copied by hand weeks earlier — exactly the step README documented
(`cp ctxoptimize/cljgo.json ~/ctxoptimize/grammars/cljgo.json`). The pack was
correct, generated, committed, and inert; the only thing that ever made it work
was a manual copy nobody else had performed.

## Decision

Generate to **`.ctxoptimize/grammars/cljgo.json`** — the path that is read, and
read *first*.

One line in `tools/gen-editors.js`. The pack is still generated from
`definers.json` per [ADR 0002](0002-definers-single-source.md), still committed,
still reviewable in a diff. It simply lands where the reader looks.

### The wasm is NOT committed

`.gitignore` already carries `*.wasm`, and that stays. This repo does not commit
build artifacts, and the parser wasm is one — 1.8MB, regenerated from
`grammar.js` on every grammar change, and useless to review in a diff.

So the split is:

| file | provenance | in git |
|---|---|---|
| `.ctxoptimize/grammars/cljgo.json` | generated from `definers.json` | **yes** — it is the reviewed decision |
| `.ctxoptimize/grammars/cljgo.wasm` | compiled from `grammar.js` | no — `*.wasm` is ignored |

Which means a fresh clone needs **one** command to become self-gathering, and
`tools/ctx-pack.sh` is that command: it runs `ctx-optimize languages add` on this
repo (which builds the wasm), then drops the wasm beside the committed JSON.

Committing the wasm was the alternative. Rejected: it contradicts a `.gitignore`
rule this repo already made deliberately, and it would put a 1.8MB binary into
history on every grammar edit.

## Verified

With the pack at the new path and the wasm built beside it, against an **empty**
machine-wide grammars dir — so nothing but this repo's own files can be
responsible:

| | |
|---|---|
| `.cljg` files | 7 |
| declarations | 20 function, 22 variable, 7 module |
| nodes named after a defining macro | **0** |
| call sites emitted as declarations | **0** |

`defcommand` is among them, which is the point of [ADR 0002](0002-definers-single-source.md):
a definer cljgo creates at run time, which no upstream registry can know, and
which this repo declares for itself.

## Consequences

- `ctxoptimize/` is removed. Anything referencing it is stale — README's
  dataflow diagram and its `cp` line, and ADR 0003's verification block.
- The documented user flow no longer needs a hand copy for *this* repo. A
  consumer project still copies the pack (or its own extended version) into its
  own `.ctxoptimize/grammars/`, which is unchanged and remains the supported path
  for project-local definers.
- `node tools/gen-editors.js --check` keeps guarding staleness, now against the
  real path, so this cannot silently regress.

## Not claimed

That upstream ctx-optimize should learn to read `ctxoptimize/`. It should not:
that would standardise a directory name one repo guessed. If a grammar repo is to
ship a pack for auto-discovery, the contract belongs in ctx-optimize's own docs
first — tracked there as
`openspec/changes/2026-07-25-auto-identify-packs`, still a draft.
