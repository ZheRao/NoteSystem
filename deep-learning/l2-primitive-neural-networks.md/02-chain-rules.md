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

