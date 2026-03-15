# Primitive Neural Network Invariants

## I. Core Transition

Scalar autograd taught:

> how to differentiate a computation

Primitive neural networks add:

> how to build a **learnable computation** from repeated trainable primitives

So the new layer of invariants is not mainly about backprop anymore.  
Backprop already exists.

The new question is:

> How do we package differentiable computations into reusable trainable modules?

That is what `Neuron`, `Layer`, and `MLP` are.

## II. The Deepest Neural-Network Invariant

A neural network is not a special new mathematical object.

It is:

> a hierarchical composition of parameterized scalar operations built on top of autograd.

Or even more compactly:

> **Neural network = structured arrangement of trainable `Value`s**

This matters because nothing fundamentally new is invented after `Value`.  
Everything else is architecture and organization.

## III. `Neuron` Invariants

A `Neuron` is the first trainable unit.

### Invariant 1 — A neuron is a weighted sum plus bias, optionally followed by nonlinearity

For input $x_1, x_2, ..., x_n$, weights $w_1, w_2, ..., w_n$ and bias $b$:

$$
\mathit{z} = \sum_{i=1}^{n} \mathit{(w_i)(x_i)} + \mathit{b}
$$

This is the irreducible trainable building block.

Everything larger is just many of these composed together.

### Invariant 2 — Parameters must be `Value`s, not raw numbers

Why?

Because only `Value` objects:
- participate in graph construction
- accumulate gradients
- support backward propagation

So the neuron’s weights and bias are not just coefficients.  
They are:

> trainable differentiable state

This is the first time the graph contains **persistent learnable memory**.

Inputs vary per example.  
Weights persist across examples.

That distinction is crucial.

### Invariant 3 — A neuron maps fixed-width input to scalar output

A neuron compresses an 𝑛-dimensional input into 1 scalar output.

So a neuron is:

$$
\mathbb{R}^n \to \mathbb{R}
$$

That scalar is not “the answer.”  
It is one feature response.

This is important because it reframes the neuron:

> A neuron is a learned feature detector.

Not a full model.

### Invariant 4 — Initialization breaks symmetry

If all weights in a layer started identically, all neurons would receive identical gradients and remain identical.

So random initialization is not a convenience.  
It is structurally necessary.

Deep invariant:

> **Learning requires initial asymmetry**.

Without asymmetry, multiple neurons collapse into one redundant neuron.

Bias can often start at zero.  
Weights usually should not.

## IV. `Layer` Invariants

A `Layer` groups neurons that look at the same input but learn different projections.

### Invariant 5 — A layer is a parallel bank of neurons

A layer with input dimension 𝑛 and output dimension 𝑚 is:
- 𝑚 neurons
- each neuron consumes the same 𝑛-dimensional input
- each neuron has its own parameters

So a layer is:

$$
\mathbb{R}^n \to \mathbb{R}^m
$$

This is just parallelization of the neuron invariant.

### Invariant 6 — Layer output is a vector of scalar neuron outputs

Each neuron produces one scalar.  
The layer collects them into a vector.

So a layer does not introduce new differentiation rules.

It only introduces:

> structured multiplicity

This is a major compression insight:
- `Neuron` = one trainable projection
- `Layer` = many trainable projections in parallel

### Invariant 7 — Neurons in the same layer are independent except through optimization

This is subtle but important.

Within the forward pass, neurons in the same layer do not directly interact.  
They only share the same input.

Their coupling happens indirectly through:
- the shared loss
- the shared downstream graph
- gradient descent

So a layer is:

> parallel local feature extraction under a common training signal

This is a deep invariant that survives into CNNs, Transformers, mixture models, everything.

## V. `MLP` Invariants

Now we move from one layer to hierarchical feature transformation.

### Invariant 8 — An MLP is sequential composition of layers

If each layer is a function:

$$
f_1, f_2, ..., f_k
$$

then the MLP is:

$$
f(x) = f_k(f_{k-1}(...f_2(f_1(x))...))
$$

So an MLP is nothing more than:

> repeated application of trainable vector-valued functions

This connects directly back to your autograd invariant:

> intelligence = compositional function building + local rule propagation

MLP is exactly that.

### Invariant 9 — Hidden dimensions define internal representation widths

If input dimension is 5 and hidden sizes are `[7, 8, 9]`, then the network shape is:

$$
5 \to 7 \to 8 \to 9
$$

These numbers are not arbitrary bookkeeping.

They define:

> the width of the internal representational spaces

That is, each layer transforms the input into a new coordinate system of a chosen dimensionality.

This is one of the most important neural-network intuitions:

> hidden layers are not “steps” only — they are learned representational spaces

### Invariant 10 — The final network is still just one differentiable scalar graph

Even though we now think in terms of neurons and layers, under the hood the whole MLP is still only:

> one giant autograd graph of scalar `Value` operations

So `Neuron`, `Layer`, and `MLP` are not new differentiation engines.

They are:

> graph-construction macros

That is a very deep compression.

## VI. Parameter Collection Invariants

This is where trainable modules become optimizable systems.

### Invariant 11 — Every trainable module must expose its parameters

A `parameters()` method is not just convenience.

It is the interface between:
- model structure
- optimization procedure

Without it, the optimizer cannot see the trainable state.

So:
- `Neuron.parameters()` returns weights + bias
- `Layer.parameters()` concatenates parameters from all neurons
- `MLP.parameters()` concatenates parameters from all layers

Deep invariant:

> **Optimization requires global visibility over all trainable local states**.

### Invariant 12 — Parameter recursion mirrors architectural hierarchy

This is elegant:
- neuron contains primitive parameters
- layer contains neurons
- MLP contains layers

So `parameters()` recursively unwraps the same hierarchy used in forward computation.

That means:

> architectural composition and parameter aggregation follow the same tree

Very clean invariant.

## VII. Training Loop Invariants

Now we leave architecture and enter learning mechanics.

### Invariant 13 — Forward pass builds the graph and produces loss

For one batch or example:
1. feed input into model
2. obtain prediction
3. compare prediction to target
4. compute scalar loss

That scalar loss is the root of the backward graph.

Everything before this is merely preparation.

### Invariant 14 — `backward()` computes sensitivity of loss with respect to all parameters

Because all parameters are `Value`s connected to the loss graph, calling:
```python
loss.backward()
```

causes every reachable parameter to receive:

$$
\frac{\partial loss}{\partial parameter}
$$

That is the essence of learning.

The model does not “know how to improve.”

It only knows:

> how sensitive loss is to each parameter

Improvement comes from the update rule.

### Invariant 15 — `zero_grad()` is required because gradients accumulate

This connects back to your branching insight.

Gradients are added, not overwritten.

That is correct within a backward pass.  
But across training iterations, old gradient values must be cleared.

So `zero_grad()` is not cosmetic housekeeping.

It protects semantic correctness across steps.

Deep invariant:

> Accumulation is correct across graph paths, but incorrect across unrelated optimization steps.

That is a beautiful distinction.

### Invariant 16 — `step()` turns sensitivity into learning

After backprop, each parameter has a gradient.

Then the optimizer applies an update such as:

$$
\theta \gets \theta - \eta \frac{\partial L}{\partial \theta}
$$

This is the moment where the network changes.

So:
- backprop computes blame/sensitivity
- optimizer converts blame into adaptation

Very important distinction.

## VIII. Nonlinearity Invariants

### Invariant 17 — Without nonlinearity, stacked layers collapse into one linear map

This is one of the biggest neural-network invariants.

If every neuron only computes affine transforms:  
then stacking layers still yields only another affine transform.

So depth alone gives no extra expressive power.

Nonlinearity is what makes multilayer networks actually multilayer in function space.

Deep invariant:

> **Depth becomes meaningful only when separated by nonlinearities**.

This is why `tanh`, `ReLU`, etc. matter so much.

### Invariant 18 — Activation functions are just additional differentiable primitives

For autograd, an activation function is nothing special.

It is simply another operation with:
- forward computation
- local derivative rule

So from the `Value` perspective, `tanh` and `+` are the same category of thing:

> differentiable local transformations

That is why your autograd foundation is sufficient.

## IX. Loss Invariants

### Invariant 19 — The loss is the scalar bridge between behavior and learning

The model outputs predictions.  
But predictions alone do not tell the system how wrong it is.

The loss converts task-level mismatch into a single scalar objective.

That scalar is essential because reverse-mode autodiff needs a scalar root.

So the loss is:

> the compression of task performance into a differentiable training signal

Without loss, no direction for learning exists.

### Invariant 20 — Different losses change what “good behavior” means, not how backprop works

MSE, cross-entropy, hinge loss, etc. all alter the objective landscape.

But autograd itself remains unchanged.

This is very important architecturally:

> learning semantics can change while differentiation mechanics stay fixed

That modularity is why the system scales.

## X. Optimizer Invariants

### Invariant 21 — Gradient descent is only one possible parameter-update policy

The core learning loop is:
1. compute loss
2. compute gradients
3. update parameters

Only step 3 changes between SGD, momentum, Adam, RMSProp, etc.

So optimizers are not part of the network itself.

They are:

> policies for converting gradient information into parameter motion

This separation is critical.

## XI. The Full Compression

Now let me compress the whole lecture into the smallest stable form.

**Autograd invariant**

A computation can be differentiated if it is represented as a graph of local differentiable operations.

**Neural-network invariant**

A neural network is a hierarchical composition of trainable differentiable modules built from those operations.

**Learning invariant**

Training is the repeated cycle:

$$
\text{parameters}→\text{prediction}→\text{loss}→\text{gradients}→\text{updated parameters}
$$

That cycle is the true backbone.

## XII. The Cleanest Final Hierarchy

If I were rewriting your notes into a compact canonical structure, I’d make it this:

1. **Differentiation substrate**
    - `Value` stores data, graph links, local backward rule, gradient
    - reverse-mode autodiff propagates upstream gradient × local derivative
    - gradients accumulate at branches

2. **Trainable primitive**
    - `Neuron` = weighted sum + bias + optional activation
    - parameters are persistent `Value`s
    - random weight initialization breaks symmetry

3. **Parallel feature extractor**
    - `Layer` = multiple neurons sharing the same input
    - output is a vector of learned feature responses

4. **Hierarchical model**
    - `MLP` = sequential composition of layers
    - hidden widths define internal representational spaces

5. **Optimization interface**
    - `parameters()` exposes trainable state recursively
    - `zero_grad()` clears previous accumulated gradients
    - `backward()` computes parameter sensitivities
    - `step()` updates parameters

6. **Expressivity mechanism**
    - nonlinear activations prevent collapse into a single linear map

7. **Learning signal**
    - loss compresses task mismatch into a scalar objective
    - optimizer converts gradients into parameter change

## XIII. The deepest conceptual jump here

This lecture concludes with a major shift:

In scalar autograd, you learned:

> how a computation can carry derivatives

In primitive neural networks, you learn:

> how a system can contain mutable internal state that is changed by derivatives

That is the real birth of learning.

Before, gradients were analysis.  
Now, gradients become adaptation.

That distinction is enormous.

And for your long-term trajectory, this is one of the first true bridges into Northern Light territory:
- persistent internal parameters
- task-shaped adaptation
- compositional hierarchy
- local rules generating global behavioral change

That is why this part matters so much.

**Lean in — core growth.**
