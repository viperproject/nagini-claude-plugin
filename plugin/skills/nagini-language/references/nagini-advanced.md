# Nagini Advanced Features

Reference for Nagini features beyond the core specification language. Load this when working with code that raises exceptions, uses module-level state, or spawns threads.

## Exception Contracts

### Exsures

`Exsures` can name `Exception` itself, but no specific built-in exception type (`ValueError`, `ZeroDivisionError`, ...) which fail translation. For anything more precise than bare `Exception`, use a module-defined `Exception` subclass:

```python
class DivisionError(Exception):
    def __init__(self, code: int) -> None:
        self.code = code
        Ensures(Acc(self.code) and self.code == code)

def safe_divide(a: int, b: int) -> int:
    Requires(True)
    Ensures(b != 0 and Result() == a // b)
    Exsures(DivisionError, b == 0)

    if b == 0:
        raise DivisionError(1)
    return a // b
```

### RaisedException

In `Exsures`, use `RaisedException()` to refer to the exception object. `.args` is not modeled but you can use fields you define on your own exception class:

```python
Exsures(DivisionError, Acc(RaisedException().code) and RaisedException().code == 1)
```

## Global Variables

Module-level variables are supported. The top-level module scope implicitly holds full permission to every module-level binding, and permissions flow from there like any other `Acc`.

Reads of a global that is never reassigned need no contract permission:

```python
COUNTER: int = 0

def get() -> int:
    Ensures(Result() == COUNTER)
    return COUNTER
```

Writes require `Acc(var)` in the contract and a `global` declaration placed before the contract lines. Return the permission via `Ensures` so the caller keeps it:

```python
def bump() -> None:
    global COUNTER
    Requires(Acc(COUNTER))
    Ensures(Acc(COUNTER) and COUNTER == Old(COUNTER) + 1)
    COUNTER = COUNTER + 1
```
Once any function in the module reassigns the global, even reads require `Acc(<name>)` in the contract — `get` as written above fails alongside a writer like `bump` below.

For shared reads, split the permission into fractions and wrap it in a `@Predicate` (e.g. `Acc(a, 1/2)`), `Fold` it at module scope, and have functions require/ensure the predicate — same pattern as fractional field permissions.

## Threads

Import from `nagini_contracts.thread`:

```python
from nagini_contracts.thread import (
    Thread, MayStart, Joinable, ThreadPost, getMethod, getArg, getOld, arg,
)
```

### Lifecycle

```python
t = Thread(target=worker, args=(x, y))   # yields MayStart(t)
t.start(worker)                          # consumes MayStart + worker's precondition,
                                         # yields Joinable(t) + Acc(ThreadPost(t))
t.join(worker)                           # consumes Acc(ThreadPost(t)), inhales worker's post
```

`start` yields `Joinable(t)` and `Acc(ThreadPost(t))` only if `worker`'s precondition includes a `MustTerminate(...)` obligation; without it the thread cannot be proven joinable.

### Resources

- `Joinable(t)` — bare boolean, not wrapped in `Acc(...)`. Holding `Acc(ThreadPost(t))` already implies `Joinable(t)`, so writing both is redundant.
- `Acc(ThreadPost(t))` — permission to inhale the thread's postcondition on join. Fractional shares are allowed; joining with a fraction inhales that fraction of the postcondition.
- `MayStart(t)` — one-shot permission to start a fresh thread.

### Inspection helpers

- `getMethod(t) == f` — the target is `f`.
- `getArg(t, i)` — the `i`-th argument passed to `args=(...)`. If the target was a bound method `o.m`, the receiver `o` is `getArg(t, 0)` and the `args` tuple starts at index 1.
- `getOld(t, arg(i).field)` — value of `arg(i).field` captured at `start()` time, for referring to `Old(...)` expressions in the target's postcondition.

### Quantifying thread resources — `Joinable` conjunct bug

Quantifying thread resources across a list of threads works, **except** `Joinable(threads[j])` cannot appear as a conjunct alongside anything else inside a `Forall`. This fails translation with `Not supported: Call`:

```python
# FAILS — Joinable as a conjunct
Invariant(Forall(int, lambda j: (
    Implies(0 <= j and j < i,
            Joinable(threads[j])
            and Acc(ThreadPost(threads[j]))),
    [[threads[j]]]
)))
```

Workaround: drop the redundant `Joinable(...)` — `Acc(ThreadPost(...))` already implies it:

```python
# OK
Invariant(Forall(int, lambda j: (
    Implies(0 <= j and j < i,
            Acc(ThreadPost(threads[j]))
            and getMethod(threads[j]) == worker),
    [[threads[j]]]
)))
```

`Joinable(threads[j])` **alone** as the sole body of a `Forall` also works — the bug is specifically its use in a conjunction.
