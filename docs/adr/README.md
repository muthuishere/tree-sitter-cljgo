# ADRs

| # | title | status |
|---|---|---|
| [0001](0001-adopt-dont-fork.md) | The grammar stays stock Clojure syntax | accepted, permanent |
| [0002](0002-definers-single-source.md) | `definers.json` is the single source of truth | accepted |
| [0003](0003-language-add-decl-rules.md) | Language add: the `decl_rules` contract | accepted |

They answer one question in three parts: **if the grammar has no declaration
node, how does anything find a definition?**

0001 says the grammar will never have one, and why that is permanent.
0003 says how ctx-optimize finds them anyway — match the head symbol, take the
name from the next element — and what that deliberately misses.
0002 says why the list of head symbols lives in exactly one file.
