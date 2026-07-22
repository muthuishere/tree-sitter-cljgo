# tree-sitter-cljgo

A [Tree-sitter](https://tree-sitter.github.io/) grammar for **cljgo**, a
dialect of Clojure.

For now the grammar is identical to Clojure's syntax (derived from
[`tree-sitter-clojure`](https://github.com/sogaiu/tree-sitter-clojure)),
renamed to `cljgo` and generated for the `.cljgo` file extension. As the
dialect diverges, its distinctive forms will be added to `grammar.js` and
the parser regenerated.

## Use with ctx-optimize

```sh
ctx-optimize languages add https://github.com/muthuishere/tree-sitter-cljgo
# ensure the pack maps your files — ~/ctxoptimize/grammars/cljgo.json:
#   "exts": [".cljgo", ".cljg"]
ctx-optimize add .
```

## Regenerate after editing the grammar

```sh
tree-sitter generate
```

## Lineage

Forked from `sogaiu/tree-sitter-clojure` (same license). Credit to its
authors for the Clojure grammar this dialect builds on.
