---
name: spec-quality
description: Specification quality principles for Nagini verification. Covers predicate design, correctness properties, and method contract completeness. Use when designing or reviewing specifications for strength and completeness.
---

Specifications here does not refer to loop invariants, lemmas or intermediate assertions. These are part of the verification of the implementation. Specifications refer to the method contracts (pre/postconditions) and the predicates and pure functions used in those contracts.

# Specification Quality

There are two main parts to good specifications:
- A good vocabulary (predicates, pure functions) that enables reasoning and proofs
- Strong, complete specifications that capture the intended behavior and properties of the program using that vocabulary

# Vocabulary

Designing the vocabulary depends heavily on the problem domain. It often includes:

- **Abstract state model**: which mathematical object (`PSeq`, `PSet`, `int`, ...) each data structure is abstracted to, and how concrete state maps to it (see Pure Functions)
- **Permission footprint**: which predicates own which parts of the heap, and where fractional permissions are needed (see Predicates)
- **Structure invariants**: properties that hold across all methods (e.g. sortedness of an internal list) — decide once where they are stated, in a predicate or over the pure representation, rather than repeating them in every contract

## Pure Functions

### Purification
In most cases, it is better to reason over mathematical representations of objects rather than the heap structures directly. For example, a linked list can be represented as a mathematical sequence.

- If possible, there should be a way to map the heap structure to a pure mathematical representation. If using built-in data structures, this is often provided (and you should always use it if available). If not, you may need to write your own pure function to do this mapping.
- All remaining properties should be stated in terms of the pure representation, not the heap structure. This allows for clearer specifications and easier proofs.

### Stating properties
Once a pure representation is available, most key functional properties (e.g. sortedness) can be stated in pure terms. Getting the functions properties right is discussed below.

### Prefer built-ins over custom pure-function wrappers

Nagini ships verified contracts for many Python built-ins (`abs`, `max`/`min`, `len`, `x in xs`, `PSeq` operations). Use them directly in specs — do not write custom `@Pure` helpers that duplicate them (see `nagini-language/references/nagini-language.md`).

Only write a custom pure function when no built-in covers the operation.

## Predicates
If using built-in data structures, this is usually not necessary. If designing your own data structures, then this is critical.

There are two options:
- Predicates bundle permissions only, with all functional properties stated in pure functions over the pure representation
- Predicates also capture functional properties (e.g., a `SortedList` predicate that includes both the structural invariant and the sortedness property)

The first approach is usually easier.

# Complete Correctness Properties
You need to understand:

- What are the correctness properties of the program?
- How can we express them using the vocabulary?

For example, for common algorithms:

- **Search**: returns the correct index when found, signals absence correctly when not
- **Sort**: output is sorted AND is a permutation of the input
- **Insert/delete**: element present/absent after operation, size changes by exactly one, other elements unchanged

## Method Contract Completeness

You should ensure for each method/function:

- **Return value**: fully characterize the result, not just a weak bound (e.g., `result == len(xs)` not just `result >= 0`)
- **Functional correctness**: capture the key algorithmic property
- **Termination**: `Decreases()` for recursive functions, `MustTerminate` for methods/loops
- **Frame conditions**: state what was NOT modified — `Ensures(x.f == Old(x.f))` for untouched fields, length and values of untouched collections, etc.
- **Permissions returned**: all permissions acquired in preconditions must be returned in postconditions (unless deliberately consumed)
- **Branch coverage**: all branches of conditionals should be covered by the postcondition, not just the happy path
- **Uncallable preconditions**: avoid preconditions that are so strong (or even unfeasible) that no call site could satisfy them
- **Triggers on quantified properties**: every `Forall` in a contract needs a well-chosen trigger.