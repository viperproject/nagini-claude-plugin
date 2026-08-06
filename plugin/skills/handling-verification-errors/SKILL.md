---
name: handling-verification-errors
description: Nagini debugging and error handling reference. Provides strategies for interpreting Nagini error messages, diagnosing verification failures, and iterating on specs and code to fix issues. Use this skill when you encounter verification errors in Nagini and need to understand and resolve them.
---

# Guiding Principle

- **Never** make assumptions about the cause of a verification error without evidence.
- Instead, always use the systematic techniques described here to gather information and isolate the true cause.
- Then, you can confidently fix the underlying issue.
- **Never** weaken or delete a contract just to make verification pass; surface the mismatch instead.

# Verifier Limitations

Do not make assumptions about how the verifier works internally. If you make incorrect assumptions, you will make mistakes. The verifier is reasonably powerful. It should practically always be possible to verify code with the correct specifications. If you think you have a correct specification but the verifier cannot prove it, you are likely missing a necessary intermediate assertion or lemma.

There is a list of known verifier limitations in the `nagini-language` skill's capabilities reference. If you think you've encountered a new limitation, confirm it before acting on it:

*Never* assume the verifier has a limitation or bug without first testing it. It is very easy to jump to incorrect conclusions about the reason of a failure. Instead, always try to write a minimal self-contained case that would fail if the limitation exists and verify it. If it verifies, the limitation is not real and the error has another cause.
If you do encounter a real verifier limitation that prevents verification of correct code, report it prominently in your output with a description and the minimal snippet that demonstrates the issue.

# Verifier options

The `mcp__nagini__verify_*` tools and their parameters (per-method selection, `counterexample: true`, `include_viper: true`, `viper_args`) are documented in `nagini-language: Verification tools`; always verify one method per call. Read results from the structured response: `message` and `code` identify the error kind, `reason` the failing assertion, `startLine` the location, `counterexample`, `branchConditions` and `debug` the debugging signals. `translationFailed: true` marks syntax/type/translation errors rather than verification failures.

# Dealing with Errors

1. **Phase 1: Isolate** — transform the error into a single failing assertion.
2. **Phase 2: Diagnose** — locate the missing step and reduce it to a self-contained candidate file.
3. **Phase 3: Fix** — verify the candidate file and apply the fix to the original code.

## Phase 1: Isolate the error

Transform the error into a failing assertion.

**For method calls**: Assert the preconditions of the callee before the call to find out which one fails.

**For loop invariants**: Assert the loop invariants, either before the loop or at the end of the loop body, to determine which invariant clause is not being established or preserved.

**For branches/multiple returns**: Do *not* assume which branch is the problem. Add asserts to each branch / before each return to find out which one fails.

**For a whole-run timeout** (`TimeoutOccurred`, no position, no diagnostics — the one failure that arrives without a located error): localize manually by commenting out proof obligations until the run completes, or insert `Assert(False)` before a suspect obligation to confirm the run reaches it.

**Separate conjunctions**: If the error occurs for a conjunction of properties, determine which clause is failing:
- Multiple postconditions/invariants: assert each individually
- Assertions with multiple conjuncts: split into separate asserts
- Method calls with multiple preconditions: assert each precondition separately before the call

## Phase 2: Diagnose
Once Phase 1 has reduced the failure to a single failing assertion, diagnose **what** the solver cannot derive. Diagnosis has two stages:

1. **Locate the missing step** — evidence-guided probing: the `debug` payload classifies the failure and tells you where to aim; `Assert` probes measure what the solver can actually derive and add the missing facts.
2. **Reduce to a candidate** — escalation for the hard cases only: when the missing fact resists inline asserts, build a minimal, self-contained Python file that captures it, and try to verify it in isolation.

Of course, it is also always possible that the state (in particular `assumptions` and `state`) genuinely does not entail the fact you are asserting. No amount of solver help can fix that; it usually requires changes to contracts or loop invariants. 

### Locate the missing step
Read the evidence first: `reasonUnknown` and the goal term tell you what kind of missing step you are hunting and where to aim your probes.
If a failing diagnostic has no `debug` field, the server was launched without SMT-state collection; pass `viper_args: ["--smtStateOnError", "--assertTimeout=2000", "--reportReasonUnknown"]` yourself.

Each failing diagnostic's `debug` object contains the symbolic state at the failure, expressed in the verifier's internal term language:

| Field | Content | Use it to |
|---|---|---|
| `failedAssertion` | The exact goal term the solver could not prove | See the obligation as the solver sees it (after encoding), not as you wrote it |
| `reasonUnknown` | Why the solver returned unknown (see table below) | **Choose the fix strategy** |
| `assumptions` | All path-condition terms in scope at the failure | Scan for gross absences and anomalies. To test whether a specific fact is available, probe it with `Assert` — presence in this list is neither necessary nor sufficient for derivability |
| `branchConditions` | The branch decisions leading to the failing path | Identify which control-flow path fails |
| `state.store` / `state.heap` / `state.oldHeaps` | Local variables and heap chunks with their symbolic values | Trace which symbolic value a variable holds; spot havocked (freshly re-assigned) values after calls |
| `preambleAssumptions` | Background axioms (domains, function definitions) with descriptions | Check what definitional/typing axioms exist for the functions in the goal |
| `functionDecls` / `macroDecls` | Declared symbols | Map term names back to program entities |
| `proverEmits` | The full SMT session (declarations + assertions) as sent to the solver | Deep inspection; note it contains the CONTEXT only, not the goal query itself |

Reading terms: `x@3@05` is a symbolic constant for program variable `x` (numbers are internal versions — successive assignments create new versions). Integers are boxed: `__prim__int___box__`/`int___unbox__` wrap between Python ints and SMT ints, and `_checkDefined(_, x, id)` wraps variable reads (it is identity on the value). `QA x :: body` is a universal quantifier. Pure functions appear applied to a snapshot argument first (`ipow(_, b, e)`).

#### `reasonUnknown` decides the strategy

| Value | Meaning | Strategy |
|---|---|---|
| `(incomplete quantifiers)` | E-matching gave up: the instantiation chain to the proof was never triggered (under-instantiation). More solver time will NOT help — the solver usually gives up in well under a second. | Add stepping-stone `Assert`s that materialize the intermediate terms and facts. For quantified goals, check TRIGGER VOCABULARY: do the premise quantifiers' trigger terms occur under the goal's binder? If not, add a bridging quantified `Assert` whose trigger matches the goal's vocabulary and whose body mentions the premise triggers. |
| `canceled` | The budget ran out while the solver was still actively working. The proof may be within reach. | Either apply the performance strategies below or retry with a larger `--assertTimeout` (e.g. 10000). |
| `(incomplete (theory arithmetic))` | Nonlinear integer arithmetic (products, `//`, `%` of variables) is beyond the solver. More time will not help. | Restate the proof with stepping stones that avoid division/modulo OF PRODUCTS entirely: use the Euclid identity (`a == (a // d) * d + a % d`), pure polynomial identities (products may appear; the solver normalizes them), and the bounded-multiple inference (`0 <= m * d < d` implies `m == 0`). `(k * d) // d == k` and `(k * d) % d == 0` are NOT directly provable — derive them via the chain above. |

With the failure classified, probe. Before the failing assertion, insert additional assertions naming candidate intermediate facts. Run the verifier. The first assert that fails narrows down the missing step: everything above it the solver can prove, everything below depends on this missing step. Narrow further by inserting another assert between the last passing one and the first failing one, until you have isolated a single small fact the solver cannot take.

Insert several `Assert(...)` probes at once. Every passing probe confirms a fact the solver knows; the first failing one pinpoints the missing step. A long chain finds the answer in one run instead of many — don't limit yourself to one probe at a time.

**Often the probes themselves are the fix.** A few well-placed `Assert(...)` statements give the solver the intermediate facts (or quantifier triggers) it needs to discharge the original goal — the original error simply goes away. When that happens, you are done: keep the asserts that hold the verification together and skip creating and verifying a candidate.

#### Choosing probe asserts

To pick candidate intermediate facts to probe with, use the patterns below.

**Weakest-precondition backtracking** to move a failing assert earlier. To debug a failing `assert P`, move it earlier by computing the weakest precondition over the preceding statement. Repeat until the assert passes (bug is between the two positions) or reaches method entry (precondition too weak).

| Failing pattern | Insert before the statement |
|---|---|
| `x := E;` `assert P;` | `assert P[x := E];` |
| `if B {` `  assert P; ...` `}` | `assert B ==> P;` |
| `if B { ... } else {` `  assert P; ...` `}` | `assert !B ==> P;` |
| `if B { ... } else { ... }` `assert P;` | push `assert P;` to end of **each** branch |
| `assert A == C;` across a call | chain: `assert A == B;` then call then `assert B == C;` |
| `Assert(Implies(A, B))` | `if A: Assert(B)` |
| `assert A && B;` | `assert A;` `assert B;` |
| `ensures P ==> Q` on a method | change to `requires P` `ensures Q` |
| `assert forall i \| 0 < i <= m :: P(i);` | split: `assert forall i \| 0 < i < m :: P(i);` and `assert P(m);` |
| `assert forall i \| i == m :: P(i);` | `assert P(m);` (instantiate the singleton range) |

**Partial-property probes** for specific facts the verifier may not know.

*Check permissions:*

| Check | Nagini |
|---|---|
| Field permission | `Assert(Acc(x.f))` |
| Read permission | `Assert(Acc(x.f, 1/2))` |
| Predicate held | `Assert(list_pred(x))` |

*Check values:*

| Check | Nagini |
|---|---|
| Known value | `Assert(x.f == 42)` |
| Known bound | `Assert(x.f > 0)` |
| Non-null | `Assert(x is not None)` |

*Check aliasing:*

| Check | Nagini |
|---|---|
| Non-aliasing | `Assert(x != y)` |

*Check data structure properties:*

| Check | Nagini |
|---|---|
| Length | `Assert(len(xs) == n)` |
| Element access | `Assert(xs[i] == v)` |
| Membership | `Assert(x in s)` |
| List / Sequence prefix | `Assert(s.take(i) == t)` |
| List / Sequence suffix | `Assert(s.drop(i) == t)` |
| List / Sequence concatenation | `Assert(s + t == u)` |
| List as sequence | `Assert(ToSeq(xs) == s)` |
| Set operations | `Assert(s - t == u)` |
| Multiset count | `Assert(s.count(x) == n)` |

To probe a predicate's contents: unfold, assert individual properties and nested predicates, then fold back. Always restore the predicate.

When a fold fails, assert each component of the predicate body separately (without unfolding) to find which piece is missing. When a postcondition involving a pure function over a predicate fails, unfold and check the function's recursive structure step by step.

**Permission flow tracing**: Track permissions through the method:
1. List permissions **acquired** (preconditions, unfolds, allocations)
2. List permissions **consumed** (folds, method calls, exhales)
3. List permissions **needed** (postconditions, remaining folds)
4. Check: acquired - consumed >= needed?

#### Performance strategies

For `canceled` and other performance failures:
- Replace `Exists()` with explicit witnesses.
- Split a method into multiple methods to reduce the proof obligation of each method. In particular nested loops can cause blowup, so extracting inner loops into separate methods can help.
- Hide for-all quantifiers in predicates. Reveal locally when required, then hide again.
- Keep predicates folded in loop invariants. When a loop traverses a predicate, prefer carrying the folded predicate in the invariant and using `Unfold`/`Fold` inside the loop body (or `Unfolding(...)` for value reads) over unfolding before the loop and threading the predicate's contents through the invariant.
- Add explicit triggers, make sure the triggers are performing correctly.
- Collapse nested `Forall` into `Forall2`/`Forall3`/.../`Forall6` with a joint trigger (see `nagini-language: Multi-variable Quantification`).
- Proving termination can be very expensive. Try removing the termination measures and verify partial correctness first.

### Reduce to a self-contained candidate

After locating the facts that hold at the failure site (every probe assert that passed) and the one that doesn't (the failing fact), create a **candidate lemma**: holding facts become preconditions, the failing fact becomes the postcondition, the body starts as `pass`.

**Procedure.**

1. **Build the candidate.** Create `<method>_repro.py`. Define a fresh function with:
- **Parameters:** exactly the variables referenced in its pre- and postconditions. No return value.
- **Postcondition:** the identified failing assertion.
- **Preconditions:** a selection of passing asserts in the original method at or before the failure site — if you want to use a fact that isn't yet asserted there, go assert it in the original first; if the assert passes, you may include it, if it fails, the fact does not actually hold there and is not a valid precondition. Pick the **minimal** subset of those passing asserts that you believe should suffice.
- **Body:** `pass`.
2. **Output.** Hand Phase 3 the candidate file `<method>_repro.py`. Phase 2 does not edit the original source.
3. **Log.** One log entry with `Phase: reduce`, fields: `Precondition count`.

## Phase 3: Fix

### Verify the candidate

Make the candidate verify. Apply the same probing technique as Phase 2's locate stage: add asserts and run the verifier. When the body needs real proof machinery, consult `references/proof-construction.md` for proof techniques.

There are three possible outcomes:

- **Nothing required.** The reduced context proved the failing fact for free. Promote the candidate to a lemma and apply it in the original method.
- **Just asserts.** A flat sequence of `Assert(...)` statements (or non-recursive lemma calls) makes the body verify. Try inlining them at the failure site in the original method. If that verifies, no lemma is needed — delete the candidate file. If not, promote to a lemma and call it.
- **Real proof machinery.** The body needs induction, fuel decrement, `Unfolding`, case analysis, or recursive lemma calls. Promote to a lemma and call it from the original method.

### Lemma promotion procedure

Promotion is mostly mechanical: the candidate is already a verified, lemma-shaped function. This step just gives it a permanent home.

**File conventions.** Lemmas live in a separate file `lemma_<lemma_name>.py` (same directory as the source) and are imported into the original file. If the lemma needs predicates or pure functions defined in the source file, extract those shared definitions into a `<source_file_name>_definitions.py` file first (to avoid circular imports) and have both files import from it. Check whether a `_definitions.py` file already exists before creating a new one.

**Procedure.**
1. Rename the Phase 2 candidate `<method>_repro.py` to `lemma_<lemma_name>.py`. By default the lemma is a regular method; use `@Pure` only when the lemma must be invoked from a pure context (inside another `@Pure` function, a predicate body, or any other place that admits only pure expressions).
2. Import the lemma into the source file and invoke it where the missing step is. Continue verifying the original method.

# Stuck criteria

Verification iteration is bounded; recognise stuck-ness early so the budget is not burned on the same wrong direction:

- **Same verification error on the same source line twice in a row.** The next attempt must be a Phase 2 probing step (not another fix). A repeated identical error means the previous fix did not address the actual missing step; probing will reveal where the missing step actually sits.
- **Same verification error on the same source line three times in a row.** Stop iterating. Report the missing step you have identified, the strategies tried, and your best hypothesis for why it is hard to discharge. Recommend whether the spec needs redesign, whether a proof technique is needed, or whether the limitation is a Nagini bug.

These criteria are checks on the structured log, not on intuition. If two consecutive log entries record the same error code at the same line, the next entry must have `Phase: probe`.

# Resources

## references/debugging-examples.md
Four worked debugging examples showing the full diagnosis process in Nagini/Python syntax: permission leak in loop, missing fold before return, self-framing violation, weak loop invariant. Each example includes a **quick-check pattern summary** for rapid diagnosis.

## references/proof-construction.md
Generic reference for proving lemma bodies in Nagini: proof-writing discipline (add only what the failure shows is missing; never pre-plan), proof techniques (structural induction, fuel-based induction, case analysis, proof chaining), the fuel-based recursion pattern, and the lemma catalog (content, preservation, equivalence, bound, monotonicity, stability) as a vocabulary for the kind of fact you are proving.

# Appendix: Symptom Diagnostic Table

Quick-reference for mapping a verification error or symptom to its likely cause and concrete fix. The Phase 1-2 procedure remains the disciplined approach; this table only catalogs symptom→cause→fix pairs and assumes you have already isolated the failing assertion.

| Symptom | Common causes | Fix with code edits | Fix annotations only |
|---|---|---|---|
| "Postcondition might not hold" | **Self-framing**: value postcondition appears before `Acc()` for that field; **missing Fold**: predicate postcondition needs `Fold()` before return; **weak invariant**: loop doesn't carry enough info to the postcondition | Reorder `Acc()` before value postconditions, add `Fold()` before return, strengthen loop invariants | Same; also check that `Acc()` precedes every field reference in `Ensures` |
| "Insufficient permission to access {field}" | **Predicate not unfolded**: permission is inside a folded predicate; **consumed by fold**: `Fold(pred(x))` consumed `Acc(x.field)`; **consumed by call**: callee took permission but didn't return it; **not in loop invariant**: permission dropped on back-edge | Add `Acc()` to precondition or loop invariant; add `Unfold` before access | Same |
| "Fold might fail" | Missing field permission; missing nested predicate instance; value constraint not yet established | Establish all required permissions and constraints before `Fold` | Add assertions to establish predicate body; adjust predicate definition if a component is missing |
| "Unfold might fail" | The predicate instance is not held at this point (never acquired, or already consumed) | Ensure the predicate instance exists before `Unfold` | Same |
| "Contract might not be well-formed" | Postcondition or precondition references a field without a preceding `Acc()` for it (self-framing violation) | Reorder or add `Acc()` in contracts so every `Acc(x.f)` precedes any expression that reads `x.f` | Same |
| "Predicate body might not be well-formed" | Predicate definition reads a field without `Acc()` for it | Fix the predicate definition so each field access has a corresponding `Acc()` | Same |
| Loop invariant not established (fails before loop) | Invariant is false or permissions are absent before the loop starts | Establish invariant before loop or adjust initialization | Adjust invariant to match pre-loop state; add `Fold`/`Assert` before loop |
| Loop invariant not preserved (fails at end of body) | Missing update in loop body; inductive step needs a lemma | Strengthen invariant or fix loop body | Strengthen or weaken invariant; add `Fold`/`Unfold` inside loop body |
| "Precondition might not hold" / call might fail | Caller doesn't establish what the callee requires | Establish the missing precondition before the call | Same |
| Method verifies but caller fails | Postcondition too weak — caller needs a guarantee the method doesn't provide | Strengthen postcondition | Same |
| Error disappears when unrelated code is added | Self-framing — the extra code incidentally provides a needed permission | Make the permission explicit in the contract | Same |
| "Function might not terminate" | Missing or incorrect termination measure | Add or fix `Decreases()` on the function/method | Same |
