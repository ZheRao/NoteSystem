# `defaultdict`

`defaultdict` means:

> “When a missing key is accessed, automatically create a default value for that key.”

It lives in Python’s standard library:

```python
from collections import defaultdict
```

## 1. The normal dictionary problem

With a normal `dict`, this fails:

```python
groups = {}

groups["MFL"].append(obj)
```

Because `"MFL"` does not exist yet.

So normally you write:

```python
groups = {}

if "MFL" not in groups:
    groups["MFL"] = []

groups["MFL"].append(obj)
```

`defaultdict(list)` removes that boilerplate.

## 2. What `defaultdict(list)` does

```python
from collections import defaultdict

groups = defaultdict(list)
```

This means:

> For any missing key, create a new empty list automatically.

So this works:

```python
groups["MFL"].append(obj)
```

Internally, Python does roughly this:

```python
if "MFL" not in groups:
    groups["MFL"] = list()  # same as []

groups["MFL"].append(obj)
```

So your grouping code becomes:

```python
from collections import defaultdict

company_groups = defaultdict(list)

for obj in objects:
    company_groups[obj.context.company].append(obj)
```

That is the whole core mechanic.

## 3. Why `list` has no parentheses

This is important:

```python
defaultdict(list)
```

not:

```python
defaultdict(list())
```

Because `defaultdict` wants a **factory function**, not one shared object.

`list` means:

> “Call `list()` every time a new key is missing.”

So each company gets its own separate list.

## 4. Your exact use case

```python
from collections import defaultdict

objects_by_company = defaultdict(list)

for obj in objects:
    company = obj.context.company
    objects_by_company[company].append(obj)
```

Result shape:

```python
{
    "MFL": [obj1, obj2, obj3],
    "AZ": [obj4, obj5],
    "GL": [obj6],
}
```

To convert back to a regular dictionary:

```python
objects_by_company = dict(objects_by_company)
```

Useful when returning from a function or serializing.

## 5. Common use cases

### A. Grouping items by key

```python
rows_by_company = defaultdict(list)

for row in rows:
    rows_by_company[row["company"]].append(row)
```

Use when:

> one key maps to many records.

This is your current case.

---

### B. Counting frequency

```python
counts = defaultdict(int)

for company in companies:
    counts[company] += 1
```

Because `int()` returns `0`.

Equivalent result:

```python
{
    "MFL": 10,
    "AZ": 4,
}
```

---

### C. Building sets of unique values

```python
locations_by_company = defaultdict(set)

for row in rows:
    locations_by_company[row["company"]].add(row["location"])
```

Because `set()` returns an empty set.

Good for:

```python
{
    "MFL": {"Waldeck", "Eddystone"},
    "AZ": {"AZ Farm", "Packing"},
}
```

---

### D. Nested grouping

```python
grouped = defaultdict(lambda: defaultdict(list))

for obj in objects:
    company = obj.context.company
    source = obj.context.source

    grouped[company][source].append(obj)
```

Result:

```python
{
    "MFL": {
        "qbo": [obj1, obj2],
        "path": [obj3],
    }
}
```

This is powerful for multi-level grouping.

## 6. The mental model

Think of:

```python
defaultdict(list)
```

as a dictionary with this rule:

> “Missing key? Initialize it with an empty list, then continue.”

Think of:

```python
defaultdict(int)
```

as:

> “Missing key? Initialize it with zero.”

Think of:

```python
defaultdict(set)
```

as:

> “Missing key? Initialize it with an empty set.”

## 7. Tiny warning

Accessing a missing key creates it:

```python
groups = defaultdict(list)

groups["MFL"]
```

Now `"MFL"` exists, even though you only looked it up.

So for checking existence, use:

```python
if "MFL" in groups:
    ...
```

not:

```python
if groups["MFL"]:
    ...
```

## Documentation version

```python
from collections import defaultdict

def group_by_company(objects):
    """
    Group objects by obj.context.company.

    defaultdict(list) automatically creates an empty list
    whenever a company key is encountered for the first time.
    This removes the need to manually check whether the key
    already exists before appending.

    Returns:
        dict[str, list]: mapping from company code to objects.
    """
    grouped = defaultdict(list)

    for obj in objects:
        grouped[obj.context.company].append(obj)

    return dict(grouped)
```

Core invariant:

> Use `defaultdict` when each key needs an automatically initialized container or default value.
