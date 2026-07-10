---
name: nagini-language
description: Nagini language reference, verified examples, and verification tools. Provides the syntax, permission model, and tooling needed to write and verify Nagini Python programs. Use when writing or reading Nagini specifications, for Nagini syntax questions, or when working with the Nagini verification tools.
---

# Nagini Language & Tools

This skill provides the Nagini language reference, verified examples, and the verification tool contract.

Nagini specifications are Python function calls (`Requires()`, `Ensures()`, `Acc()`, `Fold()`, etc.) from `nagini_contracts.contracts`. Nagini translates annotated Python into the Viper intermediate language and verifies with Silicon.

## Verification tools

Verification runs through the `nagini` MCP server.

- `mcp__nagini__verify_method(path, method)` — the primary tool. Always verify one method per call. `path` must be absolute. `method` is a bare function name, `ClassName.method_name`, or `ClassName`.
- `mcp__nagini__verify_snippet(code)` — verify inline code without creating a file.

Optional parameters on both:
- `viper_args: ["--timeout=120"]` — Silicon backend arguments; always pass a timeout.
- `counterexample: true` — include concrete failing variable assignments in each diagnostic.
- `include_viper: true` — return the translated Viper program as `viperProgram`. Request only when inspecting the encoding; even small files translate to hundreds of lines.

Result shape:
```json
{"success": bool, "translationFailed": bool, "duration": float,
 "diagnostics": [{"file": str, "startLine": int, "startCol": int,
                  "code": str, "message": str, "reason": str,
                  "counterexample": str, "branchConditions": [str], "vias": []}]}
```
Pass/fail is the `success` field. `translationFailed: true` marks syntax/type/translation errors as opposed to verification failures. `startLine` is 1-indexed.

`mcp__nagini__cancel(job_token)` cancels an in-flight verification; `mcp__nagini__flush_cache()` clears the verification result cache.

## Resources

### references/nagini-language.md
Core language reference: imports, contracts, permissions, predicates, pure functions, quantification, sequences, sets, multisets, built-in functions, loops, termination, container predicates, type annotations, assert/assume, let bindings, equality. **Read this first when writing Nagini code.**

### references/nagini-advanced.md
Advanced features beyond the core: exception contracts (`Exsures`, `RaisedException`), global/module-level variables, threads (`Thread`, `MayStart`, `Joinable`, `ThreadPost`, the `Joinable` conjunct bug). Load this only when working with code that raises exceptions, mutates module-level state, or spawns threads.

### references/capabilities.md
Confirmed Nagini capabilities and limitations — what the language can and cannot express, with workarounds. Consult before concluding that a property is inexpressible or that a workaround is needed; cited as the canonical authority for "is X really a Nagini limitation?" decisions.

### examples/
Working verified `.py` files. Load individual files as needed.

| File | Description | Key techniques |
|------|-------------|----------------|
| `binary_search.py` | Binary search on a sorted list | `@Pure` boolean predicate, `PSeq`/`ToSeq()`, quantified specs (`Forall`), `Acc(list_pred(a))`, pure function in loop invariant |
| `linked_list.py` | Linked list with prepend and find | `@Predicate`, `Fold`/`Unfold`, `Unfolding` in pure functions, `Decreases(pred)`, `Optional[Node]`, `is` for reference identity in `Ensures` |
| `quicksort.py` | Quicksort with partitioning | `for` loops with `Previous(item)`, fractional permissions (`Acc(..., 2/3)`), `MustTerminate`, recursive method, list concatenation |
| `sorted_list_insert.py` | Sorted list insert maintaining sortedness and uniqueness | `list.insert`, `PSeq.drop`, inductive lemma as `@Pure` function, proof by contradiction, `ToSeq` bridge pattern |


### references/viper-language.md
Viper intermediate language reference. Load this when inspecting the encoded Viper. It documents the syntax, permissions, and constructs that appear in that output.

### Upstream documentation

Official Nagini wiki: <https://github.com/marcoeilers/nagini/wiki>. **Use only as a fallback** — consult the references above first. Reach for the wiki only if a topic is unclear or missing.
