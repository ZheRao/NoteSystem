# Linux Path Basics

In Unix/Linux path semantics:

| Symbol | Meaning                       |
| ------ | ----------------------------- |
| `/`    | filesystem root               |
| `~`    | current user’s home directory |
| `.`    | current directory             |
| `..`   | parent directory              |

So:

```text
/etc/nginx
```

means:

> start from filesystem root

while:

```text
~/projects
```

means:

> start from my user home

and:

```text
projects/data
```

means:

> start from wherever I currently am

# `pathlib.Path`

 `pathlib` follows these exact filesystem semantics instead of treating paths like arbitrary strings.

That’s actually one of the reasons `pathlib` is powerful:

* it encodes operating-system path rules directly into the abstraction
* it prevents many invalid/manual string manipulations
* it preserves semantic meaning

So this:

```python
base / child
```

is not:

```python
base + "/" + child
```

It is:

> “resolve `child` relative to `base` according to filesystem rules.”

And filesystem rules say:

> absolute paths override context.

One subtle but important detail:

```python
Path("~/data")
```

does **not** automatically expand `~`.

You need:

```python
Path("~/data").expanduser()
```

because `~` expansion is traditionally a *shell feature* (bash/zsh), not an actual filesystem path primitive.

Example:

```python
from pathlib import Path

Path("~/data")
# PosixPath('~/data')

Path("~/data").expanduser()
# PosixPath('/home/zhe/data')
```

That distinction is another classic systems lesson:

* shell semantics
  vs
* filesystem semantics
  vs
* language-library semantics

They overlap, but are not identical.

You’re starting to move from:

> “I know syntax”

toward:

> “I understand the underlying path-resolution model.”

That transition is what eventually makes debugging feel intuitive instead of mysterious.
