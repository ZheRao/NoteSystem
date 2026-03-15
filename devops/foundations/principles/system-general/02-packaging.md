








# Utilizing Code From Other Repo

Yes — you should plan for that **now**, because otherwise future-you will slowly create dependency spaghetti.

My blunt take:

**Do not make** `music-intelligence` **depend directly on large chunks of** `data-platform` or `primitive-neural-networks` 
**unless those repos expose a small, stable library surface**.

That means the real question is not “how do I pip install my repos?”  
It is:  

**what code deserves to become a reusable package boundary?**

## The right principle

A repo should only be installable as a dependency if it contains one of these:
- a **clean reusable library**
- a **small stable subsystem**
- utilities with low coupling to the rest of the repo

A repo should **not** be imported as a dependency just because “some functions in there might be useful.”

That is how projects become tangled.

## What I’d recommend for your case

`primitive-neural-networks`

This one is the better candidate for reuse.

It may eventually expose things like:
- tensor/autograd core
- nn modules
- optimizers
- losses
- training utilities

That naturally fits a library boundary.

`data-platform`

This is much less obvious.

`data-platform` sounds like a compounding systems repo with source adapters, contracts, traversal logic, storage mechanics, 
and platform orchestration. Most of that likely should not become a dependency of a music project.

Maybe a few generic utilities deserve extraction later, but I would be very cautious about importing `data-platform` directly into unrelated projects.

## Best options, in order

### Option 1 — Small reusable internal packages

This is the cleanest long-term path.

Example:
- `primitive-neural-networks` exposes package `pnn`
- `music-intelligence` depends on `pnn`

Inside `primitive-neural-networks`, structure it like:
```text
primitive-neural-networks/
  pyproject.toml
  src/
    pnn/
      autograd/
      nn/
      optim/
      init/
      utils/
```

Then in `music-intelligence`:
```bash
pip install -e ../primitive-neural-networks
```

during development, or later:
```bash
pip install git+ssh://...
```
or from a private package index if you ever set one up.

This avoids code duplication and keeps one source of truth.

---
### Option 2 — Extract shared code into a dedicated core repo

If multiple repos need the same subsystem, do not keep importing from one “main” repo.

Instead extract shared mechanics into something like:
- `northern-light-core`
- `zr-core`
- `signal-core`
- `ml-systems-core`

For example:
```text
signal-core/
  src/signal_core/
    config/
    logging/
    io/
    experiment/
    serialization/
```
Then:
- `data-platform` depends on `signal-core`
- `music-intelligence` depends on `signal-core`
- `maybe primitive-neural-networks` depends on `signal-core`

This is the correct move when reuse becomes real.

---
### Option 3 — Monorepo with multiple installable packages

This is strong if you already know these projects will keep sharing internals.

Example:
```text
projects/
  primitive-neural-networks/
  data-platform/
  music-intelligence/
  shared-core/
```
Each subproject still has its own `pyproject.toml`, but you manage them together.

This helps when:
- you want one workspace
- shared refactors are common
- boundaries are still evolving

But this only makes sense if you want them operationally close.

For your current trajectory, I think you still benefit from **separate repos**, unless sharing grows much more.

## What I would not do

**Do not do raw `git clone` imports**

Meaning:
- clone repo
- mess with `sys.path`
- import random files directly

That works short-term, but it is fragile and ugly.

**Do not duplicate “just a few functions”**

That creates silent divergence.

**Do not make `music-intelligence` import deep internals like:**
```python
from primitive_neural_network.src.models.experimental.foo import bar
```

Only import from a clean top-level public API like:
```python
from pnn import Tensor, Module, Linear
```
That forces architectural discipline.

## Practical setup I’d suggest

For each reusable repo, use a modern `src/` layout and `pyproject.toml`.

Example structure
```text
primitive-neural-networks/
  pyproject.toml
  src/
    pnn/
      __init__.py
      autograd.py
      nn/
      optim/
  tests/
```

**Minimal** `pyproject.toml`
```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "pnn"
version = "0.1.0"
description = "Primitive neural network research library"
requires-python = ">=3.11"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```

Then from your future music repo:
```bash
pip install -e ../primitive-neural-networks
```
That gives editable install behavior: one source, no duplication.

## My actual recommendation for you

### Near-term

Do **not** optimize this yet.

Right now, while `primitive-neural-networks` is still forming, just keep this principle in mind:

**Anything likely to be reused should be built behind a clean package interface**.

### Medium-term

When `primitive-neural-networks` has a stable autograd/tensor core:
- give it a proper package name
- make it pip-installable
- expose a small public API

### For `data-platform`

Only extract genuinely generic parts.

Probably:
- config loader
- artifact/version helpers
- maybe logging or pipeline scaffolding

Probably not:
- domain traversal logic
- adapter mechanics
- source-specific orchestration

## Decision labels

**Lean in — core growth**: make primitive-neural-networks an installable package once the core stabilizes.

**Park — high future value**: a shared core repo, but only after at least two projects truly need the same abstractions.

**Skip — low yield**: cloning repos and importing internal files ad hoc.

The cleanest next mental model is:

**repo ≠ package**

A repo may contain a package, but only the stable reusable layer should be installed elsewhere.