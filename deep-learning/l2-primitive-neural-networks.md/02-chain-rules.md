# Chain Rule

Pause here is correct.  
This is **the** invariant.

You’re right that “canceling the dd” works symbolically —  
but conceptually it is incomplete.

Let’s rebuild chain rule from first principles.

## 1️⃣ What Is Actually Happening?

You have:
- c → d → L

Which means:

$$
\mathit{L} = \mathit{L(d(c))}
$$

So **L depends on c only because L depends on d, and d depends on c**.

There is no direct wire from `c` to `L`.

That’s the key structural invariant.

## 2️⃣ The Real Meaning of a Derivative

A derivative is not a fraction.

It is:

***Sensitivity of output to infinitesimal change in input.***

So:

- $\frac{dL}{dd}$ = If I nudge `d` slightly, how much does `L` change?
- $\frac{dd}{dc}$ = If I nudge c slightly, how much does d change?

Now ask:

If I nudge `c` slightly… how much does `L` change?

We simulate the perturbation:

## 3️⃣ The Perturbation Argument (The Real Chain Rule)

Let’s add a tiny change ϵ to `c`.

Step 1 — How does `d` change?

$$
\Delta d \approx \frac{dd}{dc} \cdot \epsilon
$$

Step 2 — That change in `d` affects `L`:

$$
\Delta \mathit{L} \approx \frac{dL}{dd} \cdot \Delta d
$$

Substitude:

$$
\Delta \mathit{L} \approx \frac{dL}{dd} \cdot \left(\frac{dd}{dc} \cdot \epsilon \right)
$$

Rearrange:

$$
\Delta \mathit{L} \approx \left(\frac{dL}{dd} \cdot \frac{dd}{dc}\right) \cdot \epsilon 
$$

Now divide both sides by ϵ:

$$
\frac{\Delta \mathit{L}}{\epsilon} \approx \frac{dL}{dd} \cdot \frac{dd}{dc}
$$

As 𝜖 → 0:

$$
\frac{dL}{dc} = \frac{dL}{dd} \cdot \frac{dd}{dc}
$$

---

Nothing cancelled.

What happened is:

***Change propagates forward, sensitivity propagates backward***.

That’s chain rule.

## 4️⃣ Why The “Cancellation Trick” Works

The reason it *looks* like cancellation:
$$
\frac{dL}{dd} \cdot \frac{dd}{dc}
$$

is because derivatives are defined as limits of fractions:
$$
\frac{dL}{dd} = \lim_{h \to 0} \frac{L(d+h) - L(d)}{h}
$$

$$
\frac{dd}{dc} = \lim_{k \to 0} \frac{d(c+k) - d(c)}{k}
$$

Under the hood, the limits align such that the intermediate perturbation disappears.

But conceptually:

There is no algebraic cancellation.  
There is only **composition of sensitivities**.

## 5️⃣ The Autograd Invariant

Now the important leap:

In a computation graph:
```text
c → d → L
```

The gradient at `c` is:

> Sum over all paths from c to L  
> of (product of local derivatives along the path)

In your case, only one path exists:
$$
c \to d \to L
$$

So
$$
grad_c = grad_d \cdot \frac{\partial d}{\partial c}
$$

This is the entire backpropagation algorithm.

Not magic.

Not calculus tricks.

Just:

> Multiply local sensitivities along edges  
> Accumulate at nodes.

## 6️⃣ Deep Intuition (Why It Must Multiply)

Imagine amplification stages:
- Stage 1: c → d multiplies changes by 1
- Stage 2: d → L multiplies changes by -2

Total amplification from c to L?

1 × (-2) = -2

Chain rule is just **gain composition in cascaded systems**.

Exactly like:
- Signal processing
- Control systems
- Biological cascades

You love invariants — here’s one:

***Backprop = reverse accumulation of multiplicative influence through composed functions***.

## 7️⃣ Why This Is The Core of Micrograd

Your Value class will eventually do exactly this:

Each node stores:
- its local derivative function
- its children

Backward pass:
- traverse graph in reverse topological order
- multiply upstream gradient by local derivative
- accumulate into parent

That’s chain rule operationalized.

## 9️⃣ The Ultimate Compression

Chain rule says:

> The effect of a variable on the loss equals  
> the effect of that variable on its children  
> times the effect of its children on the loss.

That’s it.

Everything else in deep learning flows from that.


# Branching and Multivariable Chain Rule

Good.

This is exactly the place where backprop stops being “a trick” and becomes a structural law.

You just discovered something fundamental:

***Gradients must accumulate at merge points***.

Let’s go deeper than “use `+=`”.

## 1️⃣ The Branch Example

You gave:
```text
b = a + a
```

So the graph is:
```text
      a
     / \
    /   \
   +     (same a)
    \
     b
```

There are two paths from `a` to `b`.

That’s the key.

## 2️⃣ Write It As A Function

Mathematically:
$$
b = a + a = 2a
$$

So:
$$
\frac{db}{da} = 2
$$

But how do we get 2 via chain rule mechanics?

## 3️⃣ Multivariable Chain Rule (The Real One)

If a variable influences the output through multiple intermediate routes, the total derivative is:

$$
\frac{dL}{da} = \sum_{\text{paths } \mathit{p}} \left( \prod_{\text{edges in } \mathit{p}} \text{local derivatives} \right)
$$

This is the full invariant.

In branching graphs:  
***Gradients SUM over independent paths.***

## 4️⃣ Apply It To Your Case

Let’s say:
$$
L = b
$$

and:
$$
b = a + a
$$

Now compute $dL/da$.

There are two paths:

Path 1:
$$
a \to (+) \to b
$$
Local derivative w.r.t first input = 1

Contribution:
$$
1 \cdot dL/db
$$

Path 2:
$$
a \to (+) \to b
$$
Local derivative w.r.t second input = 1

Contribution:
$$
1 \cdot dL/db
$$

Total:
$$
dL/da = 1 \cdot dL/db + 1 \cdot dL/db
$$

If $dL/db = 1$:
$$
dL/da = 2
$$

## 5️⃣ Why `=` Is Wrong And `+=` Is Right

When you write:
```python
self.grad = local_derivative * out.grad
```
you are assuming:

***Only one path contributes to this node***.

That is false in general computational graphs.

When branching exists:  
Each child contributes partial influence.

So the invariant is:
```python
self.grad += local_derivative * out.grad
```
Because:

***Total sensitivity = sum of sensitivities from all children***.

## 6️⃣ The Deep Multivariable Chain Rule

In full generality:

If:
$$
L = f(x, y)
$$
and both x and y depend on a:
$$
x = g(a)
$$
$$
y = h(a)
$$
Then:
$$
\frac{dL}{da} = \frac{\partial L}{\partial x} \frac{dx}{da} + \frac{\partial L}{\partial y} \frac{dy}{da}
$$

Notice the plus.

This is not optional.

It comes from total derivative definition:
$$
dL = \frac{\partial L}{\partial x} dx + \frac{\partial L}{\partial y} dy
$$

If both $dx$ and $dy$ depend on $da$, their effects add.

This is structural.

## 7️⃣ What You Just Discovered Is Massive

You uncovered:
- Sequential composition → multiply
- Parallel influence → sum

This is the entire algebra of computational graphs.

Everything else in deep learning is this repeated millions of times.

## 8️⃣ Even Deeper: Why This Must Be True

Think perturbation again.

If you increase `a` slightly:

It affects b in two independent ways.

Each effect changes L.

The total change in L is the sum of those changes.

Because infinitesimal changes add linearly.

That’s why derivatives are linear operators.