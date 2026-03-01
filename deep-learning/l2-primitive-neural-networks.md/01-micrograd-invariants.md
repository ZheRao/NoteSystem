# Autograd Invariants

## I. Representation Invariants (What Must Exist)

These are the structural requirements for reverse-mode autodiff to even be possible.

### 1️⃣ Computation Must Be a Graph (DAG)

Neural networks are not formulas.

They are:
> **Directed Acyclic Graphs of scalar operations.**

Each node:
- stores a scalar value
- is produced by an operation
- depends on parent nodes

This removes symbolic explosion.

We never manipulate giant expressions.  
We manipulate graph structure.

---
### 2️⃣ Differentiation Requires Graph Retention

To differentiate, we must retain:
- data
- parents
- operation context
- gradient accumulator

Hence Value must contain:

```Python
data
grad
_prev      # parents
_op        # operation (for debug/visualization)
_backward  # local derivative rule
```

This is minimal and sufficient.

Anything less → cannot backprop.  
Anything more → unnecessary coupling.

---
### 3️⃣ Operator Overloading = Graph Builder

`__add__`, `__mul__`, etc.

These do not “compute math.”

They:
1. Compute forward value.
2. Create a new node.
3. Attach parents.
4. Attach local derivative rule.

This is profound:
> **Math syntax becomes graph construction syntax.**

That’s the trick.

## II. Gradient Flow Invariants (How Learning Propagates)

Now we move from representation → mechanics.

### 4️⃣ Backprop Is Pure Chain Rule

For any node `v`:

$$
\mathit{v.grad} += \sum_{u \in children(v)} \mathit{u.grad} \times \frac{\partial u}{\partial v}
$$

Core invariant:
> **Gradient = upstream influence × local sensitivity**

That’s it.

Everything else is traversal bookkeeping.

---
### 5️⃣ Each Node Needs Only Two Things

To propagate gradients, a node only requires:
1. Upstream gradient (`out.grad`)
2. Local derivative rule (`∂out/∂input`)

This is huge.

It means:
- Nodes are locally ignorant of the global graph.
- No node needs the entire expression.
- Intelligence is locally compositional.

This is why reverse-mode scales.

---
### 6️⃣ Branching Implies Accumulation

If multiple paths flow into a node:

$$
\mathit{v.grad} += \textit{contribution}_1 + \textit{contribution}_2 + \cdots
$$

This is not optional.

This is required because:
> **Influence from multiple children must accumulate.**

This is the moment where people first “feel” reverse-mode instead of memorizing it.

## III. Local Rule Invariants (Why It Scales)

### 7️⃣ Arbitrary Complex Functions Are Allowed

Autograd does not require decomposition into primitives.

It only requires:

$$
\frac{\partial output}{\partial input_i}
$$

This means:
- `tanh`
- `exp`
- `pow`
- entire subroutines

are valid as long as local derivatives are defined.

This decouples:
> forward computation complexity  
> from  
> backward propagation complexity

This is scalability.

## IV. Closure Invariant (The Hidden Structural Trick)

This is the part that actually resolved your paradox.

### 8️⃣ `_backward()` Is a Local Gradient Capsule

Inside `__mul__`:

```python
def _backward():
    self.grad += other.data * out.grad
    other.grad += self.data * out.grad
```

This closure captures:
- `self`
- `other`
- `out`

That means:

The node carries with it:
> **Its own local gradient logic, pre-bound to its specific parents.**

No global lookup.  
No graph inspection.  
No symbolic traversal.

Each node knows *how to backprop itself*.

This resolves your earlier paradox:
> **“If we don’t store the entire computation graph centrally, how do we know what derivative to apply?”**

Answer:  
The derivative rule is attached to the node itself.

Backprop becomes:
```text
Traverse nodes in reverse topological order
Call node._backward()
```

That’s it.

The graph does not need to know math.  
Each node does.

## V. Deep Structural Compression

Now let’s compress everything.

Autograd requires exactly three invariants:

---
### Structural Invariant

A forward pass builds a DAG where each node stores:
- value
- parents
- gradient accumulator
- local backward rule

---
### Propagation Invariant

Gradients propagate by:

$$
\textit{upstream} \times \textit{local derivative}
$$

and accumulate at branches.

---
### Encapsulation Invariant

Each node encapsulates its own derivative rule via closure.

No global symbolic differentiation.  
No global math engine.

Just:  
Local rule × graph traversal.

## VI. Your Higher-Level Invariant

You wrote:
> **Intelligence = compositional function building + local rule propagation.**

That’s not poetic.  
That’s structurally correct.

In fact, this is exactly how:
- reverse-mode autodiff
- belief propagation
- message passing in graphs
- neural computation

all work.

They are:
> **Local transformations + signal propagation over structure.**

That invariant survives far beyond micrograd.

## VII. One Small Refinement (Important)

There is one invariant you have not explicitly stated yet, but it’s critical:

### 🔹 Reverse Topological Ordering Is Required

Backprop only works if:  
Children are processed before parents.

Why?

Because upstream gradients must already be computed before local rules apply.

So the real engine is:
```text
1. Build graph during forward
2. Topologically sort nodes
3. Traverse in reverse order
4. Call _backward() on each
```

Without reverse topological order → gradients break.

This is the final structural requirement.