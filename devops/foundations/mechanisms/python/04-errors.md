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