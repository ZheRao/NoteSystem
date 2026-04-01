# Tree Walker Engine Derived from QBO Nested JSON

What you want is to separate:
1. generic tree walking
2. node classification
3. context update rules
4. row emission rules

Right now those ideas are still mentally separated in you, but probably still partially fused in code. The reusable engine comes from making those boundaries explicit.

## The target architecture

Instead of “QBO flattening code,” think:
> A contract-driven hierarchical record walker

That engine should know nothing about:
- QBO
- accounts
- categories
- finance

It should only know:
- there are nodes
- nodes can be classified
- some nodes update context
- some nodes emit records
- some nodes are just structural containers
- child nodes are found through a declared child accessor

Then QBO becomes just one **adapter** on top.

## The mental model

### Layer 1 — Generic walker

Its job:
- visit a node
- classify it
- validate it
- update context
- maybe emit a row
- recurse into children

That is all.

### Layer 2 — Domain adapter

For QBO, define:
- how to classify node types
- how account context is extracted
- when context is inherited
- how a `Data` node becomes a flat row
- where child nodes live

### Layer 3 — Pipeline wrapper

This handles:
- reading files
- looping companies/files
- DataFrame assembly
- post-processing

That separation is what turns this into a reusable artifact.

## The exact abstraction to build

I would structure it around 5 functions.

### 1. `identify_node_type(node) -> NodeType`

This is your classifier.

QBO-specific.

### 2. `get_children(node) -> list[dict]`

Generic interface for descending the tree.

For QBO this probably means:
```python
node.get("Rows", {}).get("Row", [])
```
### 3. `update_context(node, node_type, context) -> context`

This decides whether the current node:
- establishes account context
- preserves inherited context
- should fail if expected account info is absent

QBO-specific.

### 4. `emit_record(node, context) -> dict | None`

If the node is a data node, flatten it into a row using inherited context.

QBO-specific.

### 5. `walk(node, context) -> iterable[dict]`

This is the generic engine.

## The core engine shape

Here is the clean architecture in code.
```python
from __future__ import annotations
from dataclasses import dataclass, replace
from enum import Enum
from typing import Any, Callable, Iterable


class NodeType(str, Enum):
    CATEGORY = "Category"
    CATEGORY_END = "Category End"
    ACCOUNT = "Account"
    DATA = "Data"
    SUMMARY_ONLY = "Summary Only"
    INCLUDE_DATA_FOR_PARENT = "Include Data For Parent"


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


@dataclass(frozen=True)
class TreeAdapter:
    identify_node_type: Callable[[dict[str, Any]], NodeType]
    get_children: Callable[[dict[str, Any]], list[dict[str, Any]]]
    update_context: Callable[[dict[str, Any], NodeType, WalkContext], WalkContext]
    emit_record: Callable[[dict[str, Any], NodeType, WalkContext], dict[str, Any] | None]


def walk_tree(
    node: dict[str, Any],
    adapter: TreeAdapter,
    context: WalkContext,
) -> Iterable[dict[str, Any]]:
    """
    Generic recursive walker:
    - classify
    - update context
    - optionally emit
    - recurse into children
    """
    node_type = adapter.identify_node_type(node)
    new_context = adapter.update_context(node, node_type, context)

    record = adapter.emit_record(node, node_type, new_context)
    if record is not None:
        yield record

    for child in adapter.get_children(node):
        yield from walk_tree(child, adapter, new_context)
```
This is the reusable core.

## Why this is strong

Because the walker now has **zero QBO assumptions**.

It does not know:
- what an account is
- what `Header` means
- what `ColData` means
- where IDs live

That all belongs in the adapter.

That means later you can reuse this for:
- QBO reports
- nested API payloads
- grant documents transformed into trees
- hierarchical config parsers
- any JSON with structural nodes + leaf records

## What the QBO adapter would look like

Now you plug your current ontology into this.

**Context definition**
```python
@dataclass(frozen=True)
class QBOContext(WalkContext):
    account_id: str | None = None
    account_name: str | None = None
    account_num: str | None = None
    category_name: str | None = None
    company: str | None = None
```
**Child accessor**
```python
def qbo_get_children(node: dict[str, Any]) -> list[dict[str, Any]]:
    rows = node.get("Rows", {})
    if not isinstance(rows, dict):
        return []
    children = rows.get("Row", [])
    if not isinstance(children, list):
        raise ValueError(f"Expected Rows['Row'] to be a list, got {type(children).__name__}")
    return children
```
**Context updater**

This is where the magic lives.
```python
def qbo_update_context(
    node: dict[str, Any],
    node_type: NodeType,
    context: QBOContext,
) -> QBOContext:
    if node_type == NodeType.ACCOUNT:
        header_data = node["Header"]["ColData"][0]
        raw_value = header_data.get("value", "")
        acc_id = header_data.get("id", "")

        # Example split if your account header is like "123456Account Name"
        acc_num = raw_value[:6] if raw_value else None
        acc_name = raw_value[6:] if raw_value and len(raw_value) > 6 else raw_value or None

        return replace(
            context,
            parent_account_id=context.account_id,
            parent_account_name=context.account_name,
            account_id=acc_id or None,
            account_name=acc_name,
            account_num=acc_num,
        )

    if node_type == NodeType.INCLUDE_DATA_FOR_PARENT:
        # Intentionally preserve current account context.
        # This is your scenario 5 logic.
        if not context.account_id and not context.parent_account_id:
            raise ValueError("Include Data For Parent encountered without inherited account context")
        return context

    if node_type == NodeType.DATA:
        # Data does not redefine context.
        return context

    if node_type in {NodeType.CATEGORY, NodeType.CATEGORY_END, NodeType.SUMMARY_ONLY}:
        # Usually preserve context unless you want category context tracked too.
        return context

    raise ValueError(f"Unhandled node type: {node_type}")
```
That is already much cleaner than threading `parent1_id`, `parent2_id`, `parent2_name`.

## Emission logic

This is where your philosophy becomes explicit:
> `Data` node does not care whether context came from 1 level up or 2 levels up.

Exactly.
```python
def qbo_emit_record(
    node: dict[str, Any],
    node_type: NodeType,
    context: QBOContext,
) -> dict[str, Any] | None:
    if node_type != NodeType.DATA:
        return None

    if not context.account_id and not context.parent_account_id:
        raise ValueError("Data node encountered without any inherited account context")

    coldata = node.get("ColData")
    if not isinstance(coldata, list):
        raise ValueError("Data node missing ColData list")

    effective_account_id = context.account_id or context.parent_account_id
    effective_account_name = context.account_name or context.parent_account_name

    return {
        "AccID": effective_account_id,
        "AccName": effective_account_name,
        "RawColData": coldata,
        "Corp": context.company,
    }
```
Later you replace `"RawColData"` with your real tabular extraction logic.

## The deepest improvement here

Your first version encoded hierarchy using:
- local recursion
- mutable variables
- fixed-depth memory

This generic engine instead encodes hierarchy as:
> context transitions across typed nodes

That is much more powerful.

Because once you think in terms of context transitions, the same engine works for any nested system where:
- sections change meaning
- leaves emit rows
- some nodes inherit parent state

That is a very general pattern.

## What I would do next, concretely

**Step 1**

Keep your current `_identify_node_type()` but harden it.

That classifier becomes the core of the adapter.

**Step 2**

Write `qbo_get_children(node)`.

Tiny, but important.

**Step 3**

Create `QBOContext` dataclass.

Stop threading multiple parent variables manually.

**Step 4**

Write `qbo_update_context()`.

This is the replacement for your old `parent1_id`, `parent2_id`, `parent2_name` logic.

**Step 5**

Write `qbo_emit_record()`.

This is the replacement for “if type == Data then build row.”

**Step 6**

Run the generic walker against one sample report and compare output with your existing flatten result.

That gives you confidence without rewriting the whole pipeline at once.

## Why this matters beyond QBO

This is not just prettier code.

This is you extracting a reusable invariant:
> many messy real-world systems are trees of structural nodes plus leaf records, and flattening them requires typed traversal plus inherited context

That pattern appears everywhere:
- report JSON
- XML documents
- policy trees
- menu/config systems
- nested grants/eligibility structures
- semi-structured API payloads

So this is real compounding architecture work.

## My blunt recommendation

Do not jump straight to “make it hyper-generic.”

Keep it at the right level:
- generic walker
- QBO adapter

That is enough.

Do not build a framework with 15 extension points.  
Build one clean walker and one clean adapter.

That is the sweet spot.

## Naming suggestion

A good internal naming structure might be:
- `tree_walk.py` → generic walker
- `qbo_report_adapter.py` → QBO-specific adapter
- `qbo_flatten.py` → pipeline wrapper

Or inside your repo:
- `core/tree_walker.py`
- `sources/qbo/report_adapter.py`
- `sources/qbo/report_flatten.py`

That would be very clean.