---
name: verifying
description: Guide to verifying Python programs with Nagini using this plugin's agents and skills. Use when the user asks to verify, prove, or formally check Python code, or to add specifications to Python code.
---

# Verifying Python with Nagini

Nagini proves Python code correct against contracts written in the code (`Requires`/`Ensures` from `nagini_contracts`). This skill maps the plugin's components onto a workflow that works well; adapt it to the task at hand.

## Checking for the required tools
Before performing any verification, check that the Nagini MCP server is installed and reachable, which means that the plugin's `mcp__nagini__verify_method` and `mcp__nagini__verify_snippet` tools are available.

If they are not, do not use workarounds like falling back to the `nagini` CLI. Instead,  diagnose with the plugin README.md and walk the user through the fix. Since the server spawns once at startup with Claude Code's launch environment, most fixes will require restarting Claude Code.

## Assess what exists

A verification task is characterized by which artifact layers already exist:

1. **Interface** — class definitions and typed method signatures
2. **Spec vocabulary** — predicates and pure functions
3. **Method contracts** — `Requires`/`Ensures` on methods
4. **Implementation** — executable method bodies
5. **Proof annotations** — loop invariants, fold/unfold, asserts, lemmas

Read the target files and determine, per layer, what is already there. Whatever exists is given: treat it as read-only and tell every agent so. If a given artifact turns out to be inadequate, report that to the user instead of changing it. The exception is proof annotations: existing invariants, folds, and asserts are generally always working material and may be revised.

Whatever is missing determines the work. Typical cases: nothing exists (design everything), code exists (infer specs, then prove), code and specs exist (add proofs), contracted stubs exist (implement and prove).

## Components

Agents:

- **spec-designer** — designs the spec vocabulary and method contracts and writes them into the file. Use when contracts or vocabulary are missing.
- **spec-critic** — read-only critique of specifications for completeness and strength. Use after new specs are written; it catches contracts that would verify but promise too little.
- **method-verifier** — implements a method body (where missing) and adds proof annotations, running the verifier until it passes. Dispatch one method at a time, callees before callers.
- **red-flag-checker** — read-only audit for patterns that undermine verification, from soundness problems (the result cannot be trusted) to avoidable flaws. Deciding what to do with the findings is yours.

Skills:

- **nagini-language** — syntax, permissions, built-ins, verified examples, and the verify-tool contract
- **spec-quality** — what makes contracts strong and complete
- **handling-verification-errors** — the debugging playbook for failing or timing-out verification

Verification runs through the `nagini` MCP tools (`verify_method`, `verify_snippet`), documented in the nagini-language skill.

## Suggested flow

1. **Specs** — spec-designer writes vocabulary and contracts for the methods that lack them.
2. **Review** — spec-critic and red-flag-checker (phase: `spec-review`), in parallel, before any proof effort is spent: fixing a weak contract now is far cheaper than after methods verify against it. Loop back to step 1 with the findings if needed.
3. **Implement & prove** — method-verifier per method, callees before callers so callers can rely on verified contracts; independent methods may be dispatched in parallel.
4. **Final audit** — red-flag-checker (phase: `final`) over the finished file. Soundness findings mean the result cannot be trusted yet.

When starting from scratch, design the interface first (directly, with the nagini-language skill): typed signatures, `@ContractOnly` stubs, and data representations Nagini handles well.

## Principles

- Never weaken or delete a contract to make verification pass; surface the mismatch instead.
- Full functional correctness is the default goal. If the user wants less (safety only, specific properties), settle that before specs are designed.
