
**Assertions**
---

```python
assert condition, "error message"
```

**Try / Except / Else / Finally**
---

```python
try:
    ...     # condition 
except Exception:
    ...     # if condition fail
else:
    ...     # if condition didn't fail
finally:
    ...     # regardless of condition
```

**Positional Arguments (`*args`)**
---

```python
def fn(x, **kwargs):
    print(x)
    print(kwargs)
```

**Type Hints (`typing`)**
---

```python 
from typing import Optional, Union

def fn(text: Optional[str]):
    ...

def fn(value: Union[str, int]):
    ...
```

**Pattern Matching (`match`)**
---

```python
match error_code:
    case 200 | 201:
        print("success")
    case 400:
        print("not found")
    case _:
        print("unknown")
```

