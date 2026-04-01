# Dataclass Intro

Good — this is exactly the right place to slow down and remove fear.  
`@dataclass` looks magical, but it’s actually very mechanical.

Let’s strip it down.

## 🧠 What `@dataclass` actually is

At its core:
> `@dataclass` just auto-generates boring boilerplate code for classes that store data

Nothing more.

## 🔍 Your example (demystified)
from dataclasses import dataclass
```python
@dataclass(frozen=True)
class WalkContext:
    """
    Generic context object.
    Domain adapters can subclass or replace this with richer fields.
    """
    account_id: str | None = None
    account_name: str | None = None
    parent_account_id: str | None = None
    parent_account_name: str | None = None

`@dataclass`(frozen=True)
class QBOContext(WalkContext):
    account_num: str | None = None
    category_name: str | None = None
    company: str | None = None
```
This is equivalent to writing something like:
```python
class QBOContext(WalkContext):
    def __init__(
        self,
        account_id=None,
        account_name=None,
        account_num=None,
        category_name=None,
        company=None,
    ):
        self.account_id = account_id
        self.account_name = account_name
        self.account_num = account_num
        self.category_name = category_name
        self.company = company

    def __repr__(self):
        return f"QBOContext(account_id={self.account_id}, account_name={self.account_name}, ...)"
```
👉 That’s it. That’s the “magic”.

## ⚙️ What the decorator actually does

When Python sees:
```python
`@dataclass`
class X:
```
It:

1. Reads your field definitions
2. Generates:
    - `__init__`
    - `__repr__`
    - `__eq__` (comparison)
    - optionally others
3. Attaches them to the class

## 🧩 Why this is useful (in YOUR system)

You are passing around context during tree traversal.

Instead of:
```python
parent1_id, parent2_id, parent2_name, company
```
You now have:
```python
context.account_id
context.parent_account_id
context.company
```
👉 This is structured state, not loose variables.

## 🧠 The key mental shift

**Without dataclass**
```python
def func(account_id, account_name, parent_id, parent_name, company):
```
Problems:
- order matters
- easy to mix up
- hard to extend

**With dataclass**
```python
def func(context: QBOContext):
```
Now:
- everything grouped
- self-documenting
- extensible

## 🔒 What `frozen=True` means

This is important.
```python
`@dataclass`(frozen=True)
```
means:
> once created, the object cannot be modified

**Example**
```python
ctx = QBOContext(account_id="123")

ctx.account_id = "456"  # ❌ ERROR
```
You’ll get:
```
FrozenInstanceError: cannot assign to field
```

## Why this is GOOD (for you)

Your system is recursive.

If context were mutable:
```python
context.account_id = "new"
```
You could accidentally:
- mutate shared state
- corrupt sibling branches
- introduce hidden bugs

**Instead, you do this**:
```python
from dataclasses import replace

new_ctx = replace(context, account_id="new")
```
This creates a new object, leaving the old one untouched.

👉 This is **functional-style safety**

## 🔁 Why this matches your architecture

Your traversal logic is:
```
node → classify → update context → recurse
```
Each step should produce:
> a new context, not mutate the old one

That’s exactly what `frozen=True` enforces.

## 🧠 What inheritance is doing here
```python
class QBOContext(WalkContext):
```
means:
> QBOContext extends WalkContext

So if WalkContext has:
```python
`@dataclass`(frozen=True)
class WalkContext:
    parent_account_id: str | None = None
```
Then QBOContext automatically has:
- `parent_account_id`
- plus your new fields

## 🧠 Strip it to intuition

Forget syntax.

Think:
> “This is just a clean container for state, with safety guarantees”

That’s it.

🧪 Tiny example (play with this)
```python
from dataclasses import dataclass, replace

`@dataclass`(frozen=True)
class Ctx:
    a: int | None = None
    b: int | None = None

ctx1 = Ctx(a=1)
ctx2 = replace(ctx1, b=2)

print(ctx1)  # Ctx(a=1, b=None)
print(ctx2)  # Ctx(a=1, b=2)
```
👉 No mutation. Clean state evolution.

## 🎯 When to use dataclass (important)

Use it when:
- object is mostly data
- no heavy behavior
- passed around frequently
- benefits from immutability

## ❌ When NOT to use it

Skip dataclass when:
- object has complex methods / lifecycle
- heavy logic inside class
- dynamic attributes

# General Dataclass

Good — this is exactly how you should approach it: **separate the concept from your current use case**.

I’ll give you a clean mental model first, then concrete patterns, then the “methods” ecosystem.

## 🧠 What `dataclass` is (general, not QBO)

Forget your current pipeline.
> A dataclass is a structured, typed container for data with predictable behavior

It sits between:
- raw `dict` (too loose)
- full class with logic (too heavy)

## 🔥 Where dataclasses are commonly used

These are the major usage patterns across real systems.

### 1. 🧾 Data transfer objects (DTOs)

Passing structured data between layers.
```python
@dataclass
class User:
    id: str
    name: str
    email: str
```
Used in:
- APIs
- ETL pipelines
- service boundaries

👉 replaces messy dicts:
```python
user["id"]  # ❌ fragile
user.id     # ✅ clear
```
### 2. 🧠 State containers (your current use case)
```python
@dataclass(frozen=True)
class Context:
    user_id: str | None
    session_id: str | None
```
Used in:
- recursion
- workflows
- pipelines
- interpreters

👉 this is your QBO case

### 3. ⚙️ Configuration objects
```python
@dataclass
class DBConfig:
    host: str
    port: int
    user: str
    password: str
```
Used in:
- system config
- environment setup
- dependency injection

### 4. 📦 Domain models (lightweight)
```python
@dataclass
class Order:
    order_id: str
    amount: float
```
Not heavy logic — just structure.

### 5. 🧪 Test fixtures
```python
@dataclass
class FakeTransaction:
    amount: float
    date: str
```
Used in:
- unit tests
- mocking

### 6. 🔄 Serialization / deserialization
```python
from dataclasses import asdict

asdict(obj) → dict
```
Used in:
- saving data
- API responses
- logging

## 🧠 Key design axis (this is important)

There are TWO major styles of dataclass usage:

### Style A — Mutable dataclass
```python
@dataclass
class Config:
    retries: int = 3
```
Used when:
- object evolves
- simple container
- no strict guarantees needed

### Style B — Immutable (frozen=True)
```python
@dataclass(frozen=True)
class Context:
    ...
```
Used when:
- correctness matters
- recursion / branching
- avoiding side effects

👉 This is closer to **functional programming**

## 🔁 Now your second question: methods ecosystem

### 1. `dataclasses.replace()`
```python
from dataclasses import replace

new_obj = replace(old_obj, field=new_value)
```
👉 creates a new instance

Used for:
- immutable updates
- state transitions

### 2. `dataclasses.asdict()`
```python
from dataclasses import asdict

asdict(obj)
```
Converts:
```python
Ctx(a=1, b=2)
→ {"a": 1, "b": 2}
```
Used for:
- serialization
- debugging
- logging

### 3. `dataclasses.astuple()`
```python
from dataclasses import astuple
```
Less common.
```python
Ctx(a=1, b=2)
→ (1, 2)
```
### 4. `dataclasses.fields()`
```python
from dataclasses import fields

fields(Ctx)
```
Returns metadata about fields.

Used for:
- introspection
- generic systems

### 5. `__post_init__` (VERY important)

This is where you add validation.
```python
@dataclass
class User:
    age: int

    def __post_init__(self):
        if self.age < 0:
            raise ValueError("Invalid age")
```
👉 runs after `__init__`

Used for:
- validation
- derived fields
- normalization

### 6. `field(...)` (advanced)

Used to customize behavior.
```python
from dataclasses import field

@dataclass
class X:
    items: list = field(default_factory=list)
```
👉 avoids mutable default bug

## 🧠 The deep connection (your intuition is correct)

***how you use dataclass determines which methods you need***

### If you use dataclass as mutable container

You mostly use:
- direct assignment
- `asdict`

### If you use dataclass as immutable state

You heavily use:
- `replace`
- `__post_init__` (validation)
- possibly `fields`

## 🔥 The real conceptual upgrade

You are touching something deeper:

**Mutable world**
```python
obj.x = 5
```
State is **changed in place**

**Immutable world**
```python
obj2 = replace(obj, x=5)
```
State is **transformed into a new version**

This leads to:
- safer recursion
- easier reasoning
- no hidden side effects

## 🧠 Why this matters beyond Python

This pattern appears in:
- functional programming (Haskell, Clojure)
- React state updates
- distributed systems
- event sourcing

## ⚠️ Common mistakes (important)

### ❌ Using mutable defaults
```python
@dataclass
class X:
    items: list = []  # BAD
```
Use:
```python
items: list = field(default_factory=list)
```
### ❌ Overusing dataclass

Don’t use it when:
- behavior dominates data
- object has complex lifecycle

### ❌ Mixing mutable + frozen thinking

Either:
- mutate freely  
OR
- use `replace`

Don’t mix both styles randomly.

## 🧠 Your current trajectory

You are naturally moving toward:
> state-transition modeling

That is:
- context flows
- no mutation
- explicit updates

That’s **high-level system thinking**

## 🎯 When YOU should use dataclass

Given your trajectory:

Use dataclass when:
- you pass structured state
- you want explicit contracts
- you want safer recursion
- you want readability

Avoid when:
- it’s just a quick script
- no structure needed
- behavior dominates

## 📌 Final compression

If I compress everything:

> dataclass = structured data + auto-generated boilerplate  
> frozen=True = enforce immutability  
> replace() = safe state transition

## 🔍 Classification (your system)
**Lean in — core growth**
- dataclass
- immutability
- state transition modeling
- `__post_init__`
**Park — high future value**
- persistent data structures
- functional programming paradigms