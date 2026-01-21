# Autograd Mastery Track

**Complementary to Linear Algebra Fundamentals**

> Objective:  
> *Internalize the engine behind neural networks so deeply that frameworks become implementation details, not authorities.*

## Phase 1 — Micrograd in Python (must-do)

**Status**: Lean in — core growth  
**Timebox**: ~5–10 focused days  
**Outcome**: You own reverse-mode autodiff

---
### Phase 1 Mental Model (anchor this first)

Autograd =

> **A graph of computations + local derivatives + reverse accumulation**

Nothing more.

You are building:
- **A scalar computation graph**
- **A topological backward pass**
- **A gradient accumulation mechanism**

---
### Phase 1 Scope (non-negotiable boundaries)

✔ Scalars only  
✔ Python only  
✔ No NumPy  
✔ No tensors  
✘ No broadcasting  
✘ No performance tricks  
✘ No “future proofing”

This phase is about **truth**, not power.

---
### Phase 1 Deliverables (externally verifiable)

A clean `Value` class
- Reverse-mode autodiff
- Tiny MLP
- XOR training demo
- README explaining why this works

---
### Phase 1 Checklist (Concrete)

**Step 1 — Scalar Value Object**  
**Goal**: Represent a scalar with gradient tracking

**Implement**
- `data: float`
- `grad: float`
- `_prev: set[Value]`
- `_op: str`
- `_backward: callable`

**Done when**
- You can manually construct `Value(2.0) * Value(3.0)`
- Graph connections are visible/debuggable

**LA tie-in**
- Scalars = 1×1 tensors
- You’re learning gradient flow *before* matrices hide it

---
**Step 2 — Operator Overloading**  
**Goal**: Build computation graph implicitly

**Implement**
- `+`, `*`, `-`, `/`
- `**` (power)
- `relu()`

Each op must:
- Create a new `Value`
- Store parents
- Store a `_backward()` closure with local derivatives

**Done when**
```python
x = Value(-4)
y = (x * 2 + 1).relu()
y.backward()
```
produces correct gradients.

**Key insight**
> Backward pass is just *the chain rule applied locally*.

---
**Step 3 — Topological Backward Pass**  
**Goal**: Correct gradient propagation order

**Implement**
- Graph traversal (DFS)
- Reverse topological ordering
- Accumulate gradients (`+=`)

**Done when**
- Any DAG computes correct gradients
- Shared subgraphs accumulate gradients correctly

**LA tie-in**
- This mirrors how matrix gradients sum contributions from multiple paths

---
**Step 4 — Backward() API**  
**Goal**: Make gradient computation ergonomic

**Implement**
```python
def backward(self):
    self.grad = 1.0
    for node in reversed(topo):
        node._backward()
```

**Done when**
- Calling `.backward()` on the output computes all leaf grads

---
**Step 5 — Tiny Neural Network**  
**Goal**: Prove learning emerges from autograd

**Implement**
- Neuron
- Layer
- MLP
No NumPy. Use Python lists.

**Train**
- XOR (binary classification)

**Done when**
- Loss decreases
- Decision boundary makes sense

**Critical realization**
> Backprop is not a neural network algorithm — it’s a *general gradient engine.*

---
**Step 6 — README (this matters)**

**You must explain**
- What reverse-mode autodiff is
- Why topological order matters
- Why gradients accumulate
- How this generalizes to tensors
This is what turns the repo into **signal**, not noise.

---
### Phase 1 Exit Criteria (be honest)

You can answer *without hesitation*:
- Why reverse-mode is efficient for NN
- Why backprop is graph-based, not layer-based
- Why gradients add instead of overwrite
If yes → Phase 1 complete.

## Phase 2 — Tensor Autograd in Python (optional but huge)

**Status**: Lean in — core growth  
**Timebox**: ~2–4 weeks (strictly scoped)  
**Outcome**: You understand framework-level autograd

---
### Phase 2 Mental Model

Phase 1 taught you:
> Autograd over scalars

Phase 2 teaches:
> Autograd over **linear algebra objects**

The leap is:
- Scalars → arrays
- Local derivatives → **Jacobian structure**
- Loops → **vectorized math**

---
### Phase 2 Scope Boundaries (do NOT violate)

✔ NumPy allowed (for storage only)
✔ Small operator set
✔ 2D tensors max
✘ No broadcasting (initially)
✘ No GPU
✘ No performance obsession

---
### Phase 2 Deliverables

- `Tensor` class with `.data`, `.grad`
- Autograd for matrix ops
- Tiny MNIST MLP (fully connected)
- Clear gradient correctness tests

---
### Phase 2 Checklist (Concrete)

**Step 1 — Tensor Object**

**Implement**
- `data: np.ndarray`
- `grad: np.ndarray`
- `_prev`, `_op`, `_backward`

**Done when**
- Tensor behaves like Value but with shape

**LA tie-in**
- Gradients now match matrix dimensions
- You *see* Jacobians implicitly

---
**Step 2 — Core Tensor Ops**

**Must implement**
- `add`
- `mul`
- `matmul`
- `sum`
- `mean`
- `relu`

Each `_backward()` must:
- Respect shape
- Accumulate gradients correctly

**Done when**
- Manual gradient checks pass

---
**Step 3 — Loss Functions**

**Implement**
- MSE
- Softmax + cross entropy (combined, numerically stable)

**Key insight**
> Loss is just another node in the graph.

---
**Step 4 — Training Loop**

**Implement**
- Parameter collection
- Zero grad
- Forward
- Backward
- SGD

**Done when**
- MNIST accuracy > random baseline
- Loss decreases smoothly

---
**Step 5 — Gradient Sanity Tests (mandatory)**

**Implement**
- Finite difference gradient check
- Compare numerical vs autodiff gradients

**If this fails, you don’t understand it yet.**

---
### Phase 2 Exit Criteria

You can:
- Explain why `d(A @ B)` produces two gradient paths
- Reason about gradient shapes without guessing
- Debug wrong gradients systematically

At this point:
- PyTorch becomes boring — and that’s the goal.

# How This Complements Linear Algebra (Very Important)

| Linear Algebra        | Autograd Track         |
| --------------------- | ---------------------- |
| Matrix multiplication | `matmul` backward      |
| Chain rule            | Backward closures      |
| Jacobians             | Implicit gradient flow |
| Vector spaces         | Tensor shapes          |
| Optimization          | SGD loop               |

You’re **connecting math → mechanism**, not just learning formulas.