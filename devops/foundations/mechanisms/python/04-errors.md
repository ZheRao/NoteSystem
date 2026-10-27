# `try` — `except` — `finally`

Good catch — this is one of those small mechanics that quietly control correctness. Let’s sharpen it cleanly.

## 🧠 Core Mental Model (No fluff)

Execution order:
```py
try:
    ...
except SomeError:
    ...
finally:
    ...
```
Think:
> try → (maybe except) → always finally

## 🔥 The 3 Rules That Matter

### 1. If `try` fails and no `except` → error propagates
```py
try:
    raise ValueError("boom")
finally:
    print("cleanup")
```
Output:
```
cleanup
Traceback (ValueError)
```
👉 `finally` runs, but error is NOT swallowed

### 2. except catches errors (and can swallow them)
```py
try:
    raise ValueError("boom")
except ValueError:
    print("handled")
```
Output:
```
handled
```
👉 No crash → error is swallowed

### 3. `finally` ALWAYS runs (even if return / error)
```py
def f():
    try:
        return 1
    finally:
        print("cleanup")

f()
```
Output:
```
cleanup
```
👉 Even `return` does NOT skip `finally`

## ⚠️ The Only Dangerous Case

This is where people get burned:
```py
try:
    raise ValueError("boom")
finally:
    return 42
```
👉 The `return` in `finally` overrides the exception

Result:
```
42   # error is LOST
```
This is the only real silent-failure trap.

## 🧩 Your Code (Why It’s Correct)
```py
try:
    ...
finally:
    if tmp.exists():
        try:
            tmp.unlink()
        except OSError:
            pass
```
What happens:
- if write fails → exception raised
- `finally` runs cleanup
- cleanup may silently fail (only unlink)
- **original exception still propagates**

👉 So:
- ❌ NOT silent failure
- ✅ fail-loud for real errors
- ⚠️ only cleanup errors are ignored

## 🧠 Minimal Truth Table
| Scenario                  | Exception? | Result                        |
| ------------------------- | ---------- | ----------------------------- |
| try succeeds              | ❌          | normal                        |
| try fails, no except      | ✅          | propagated                    |
| try fails, except handles | ❌          | swallowed                     |
| finally runs              | always     | always runs                   |
| finally has `return`      | ❌          | overrides everything (danger) |

## 🧩 Why Your Instinct Fired

You thought:
> “finally runs → does that mean errors are skipped?”

That’s a *very common misinterpretation*.

The correct model:
> `finally` runs **in addition to**, not **instead of**, error propagation.

## 🔧 When to Use What (Practical)
Use `try + finally` when:
- you want guaranteed cleanup
- but still want failure to crash

👉 your atomic write = correct use

Use `try + except` when:
- you want to recover or transform error

Use `try + except + finally` when:
- you want BOTH recovery + cleanup

## 🎯 One-line takeaway

> finally guarantees execution — it does NOT suppress exceptions unless you explicitly override them.

## 🔍 Small improvement mindset

Your current design:
- fail loudly for core operation ✅
- ignore cleanup failure ⚠️ (intentional tradeoff)

That’s a *valid engineering decision*, not a bug.


# Detailed mechanisms of Python error + context manager bug

The key idea is that **catching an exception ends it.** Once some `except` handles it, the program carries on as if nothing went wrong, and code further out never learns there was an error. I'll build up the model from the start, because the context-manager behavior and the `BaseException` question both follow from it.

## 1. What `raise` does

When Python hits an error, or a `raise` statement, it creates an **exception object** and stops executing the current line. It doesn't continue to the next line. Instead it starts **unwinding**: it leaves the current block, then the current function, then that function's caller, and so on outward, looking for a `try` with a matching `except`.

```python
def inner():
    print("inner: before")
    raise ValueError("boom")
    print("inner: after")        # never runs

def outer():
    print("outer: before")
    inner()
    print("outer: after")        # never runs either: the exception passes through

try:
    outer()
except ValueError as err:
    print("caught:", err)
print("program continues")
```

Output:

```
outer: before
inner: before
caught: boom
program continues
```

The exception travelled out of `inner`, straight through `outer` (skipping its last line), and stopped at the first matching `except`. If no `except` anywhere matches, it reaches the top of the program, which crashes and prints a traceback.

## 2. Catching ends the journey

The search stops at the **first** matching `except` on the way out. After that `except` block finishes, execution continues normally on the line *after* the `try` statement. The exception object still exists as `err` inside the handler, but it's no longer travelling. Nothing further out will see it.

To pass it further out, you must explicitly raise again. A bare `raise` inside an `except` block re-raises the same exception and resumes its journey.

## 3. Your test, as nested layers

Picture the code as layers, innermost first:

```
unit_of_work's try / except sqlite3.Error      ← outer catcher
  └── yield conn  →  your with-block body
        └── your try / except sqlite3.Error    ← inner catcher
              └── conn.execute(... duplicate ...)   ← raises IntegrityError
```

The `IntegrityError` starts at the duplicate INSERT and travels outward. The first matching `except` it meets is **yours, inside the `with` block**. It stops there, your print runs, and the body continues and reaches its end normally. From `unit_of_work`'s point of view, the body simply finished. Nothing came out of it, because the inner catcher already consumed the exception.

So the context manager didn't fail to see the error; the error never reached it. It was caught one layer closer to where it happened.

## 4. How `with` hands exceptions to a context manager

When a `with` block ends, Python tells the context manager *how* it ended. With `@contextmanager`, that becomes one of two things at your `yield` line:

- **The block ended normally:** the generator continues on the line after `yield`.
- **An exception came out of the block:** Python re-raises that exception *at the `yield` line*, inside your generator. That's why your `try/except` around `yield` can catch it.

You can watch this happen with a toy version:

```python
from contextlib import contextmanager

@contextmanager
def demo():
    print("enter")
    try:
        yield
        print("body ended normally → commit here")
    except Exception as err:
        print("exception arrived at yield:", repr(err), "→ rollback here")
        raise
    finally:
        print("finally → close here")

# Case A: exception escapes the block
try:
    with demo():
        raise ValueError("A")
except ValueError:
    print("caller saw it\n")

# Case B: exception caught inside the block
with demo():
    try:
        raise ValueError("B")
    except ValueError:
        print("caught inside the block")
```

Predict both outputs before running it. Case B is exactly what your test did.

## 5. Exception classes, and why `sqlite3.Error` vs `BaseException` matters

Every exception is an object of some **class**, and those classes form a family tree. `except SomeClass` matches that class **and every class below it in the tree**. It's an "is a kind of" check: `isinstance(err, SomeClass)`.

The relevant part of the tree:

```
BaseException
├── KeyboardInterrupt        (Ctrl+C)
├── SystemExit               (sys.exit())
├── GeneratorExit
└── Exception                (almost everything you'll meet)
    ├── ValueError, KeyError, TypeError, AttributeError, RuntimeError, …
    └── sqlite3.Error
        ├── sqlite3.InterfaceError
        └── sqlite3.DatabaseError
            ├── sqlite3.IntegrityError     (experiments E, G: unique, foreign key)
            ├── sqlite3.OperationalError   (experiments B, F: database is locked)
            └── sqlite3.ProgrammingError, DataError, …
```

Reading it:

- `except sqlite3.IntegrityError` catches only constraint violations.
- `except sqlite3.Error` catches every `sqlite3` error: integrity, operational, all of them. That's why one `except sqlite3.Error` handled both the duplicate and the lock error.
- `except sqlite3.Error` does **not** catch a `KeyError`, because `KeyError` is on a different branch. So if your service code had a bug raising `KeyError` mid-transaction, your original `unit_of_work` wouldn't run its rollback.
- `except Exception` catches every ordinary error, including all `sqlite3` errors.
- `except BaseException` catches everything, including Ctrl+C and `sys.exit()`, which deliberately sit *outside* `Exception`.

**Why `BaseException` is fine here, when it's usually discouraged.** Catching everything is normally a bad idea, because swallowing Ctrl+C or `sys.exit()` makes a program impossible to stop. But `unit_of_work` **always re-raises**. It doesn't swallow anything: it only says "whatever is leaving this block, undo the transaction first, then let it continue." For that purpose, the broadest class is the right one. If someone presses Ctrl+C mid-transaction, rolling back first is exactly what you want.

**(Lesson candidate: catch narrowly when you handle an error, and broadly only when you clean up and re-raise. A handler that consumes the exception must be specific; a cleanup step that passes it on should cover every kind of exit.)**

A portability bonus (tier 2): the `sqlite3` classes under `Error` follow Python's database standard (PEP 249). psycopg and pyodbc use the same names, `IntegrityError`, `OperationalError` and so on. So `except IntegrityError` reads the same across drivers, even though the codes inside differ.

## 6. Three small checks to run

1. **Propagation:** run the `inner`/`outer` example, then delete the `try/except` around `outer()` and read the traceback. The traceback lists the path the exception travelled.
2. **Matching:** inside a `try`, raise `KeyError("x")`, and catch with `except sqlite3.Error`. Does it catch? Then try `except Exception`.
3. **The toy context manager:** run cases A and B and compare with your predictions.

After those three, your original bug should look obvious: the database raised correctly, your test caught it first, and the unit of work committed a block that, from where it stood, had succeeded.