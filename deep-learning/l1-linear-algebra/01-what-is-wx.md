# What does *Wx* really do?

At its core:

**A matrix is a recipe for generating new features from an input**

A linear layer does not "compute" in the human sense — it re-expresses the input in a 
new coordiate system where useful structure becomes visible.

## 1. Mental geometric picture

Think of your input $\mathbf{x}$ as an object living in space.

A linear transformation can:

- **Rotate**  
    → Same information, different basis (invertible)
- **Project (flatten)**  
    → Lose dimensions (information loss)
- **Embed (extrude)**  
    → Move into a higher-dimensional description
- **Stretch / shear / skew**  
    → Emphasize some directions, suppress others
- **Learn new axes**  
    → A neural network layer chooses *its own coordinate system for understanding the data*

> A neural network layer is not learning *values* — it is learning **axes of interpretation**.

*(detail in Appendix)*

# 2. Dot prouduct = alignment

Each neuron computes

$$
w^\top_{i} x
$$

Interpretation:
- $x$ → *Raw input (image patch, sound frame, token embedding, sensor reading)*
- $w_{i}$ → *A learned direction (edge, pattern, concept, attribute)*
- $w_{i}^\top$ → *"How much does the input align with this concept?"*

High dot product means:

- Strong horizontal edge
- Shape present
- Pattern matched
- Concept detected
- Semantic alignment

Low or negative dot product means:

- No match
- Orthogonal information
- Opposite signal

> **Dot product = alignment**  
> **Alignment = feature presence**

# 3. Increasing dimensionality is not about "filling space"

Example: mapping **3D → 5D**
> "Let me feed this 3D thing into a machine that spits out a 5D description of it."

Important clarification:

- The resulting vectors **do not span all of 5D**
- That is *not the goal*

The goal is:
> **Create a space where useful features are linearly accessible**.

Higher dimensionality = more room to separate, aligh, and isolate concepts.

# Appendix: Linear Transformation Breakdown

**Definition**:

A linear transformation is **fully defined by what it does to the basis vectors**.

If:

$$
W = 
\begin{bmatrix}
\mathbf{w}_{1} & \mathbf{w}_{2} & \cdots & \mathbf{w}_{n}
\end{bmatrix}
$$

then:

$$
Wx = 
\sum_{i} x_{i} \mathbf{w}_{i}
$$

> Every linear transformation = re-expressing your input as a mixture of new axes

This sinlge fact explains *everything below*.

## 1. Rotation — *change of basis (invertible)*

**Condition**

$$
W^\top W = I \quad (orthogonal matrix)
$$

**What happens geometrically**

- Lengths preserved
- Angles preserved
- No information lost
- Space is rigidly rotated

You are not **changing the object**, only how you *look at it*.

**Mental anchor**

> Same object, different camera orientation

**Neural network meaning**

- Rarely enforced explicitly 
- But **whitened data + orthogonal layers** approximate this
- In theory: pure "re-labeling of meaning directions"


## 2. Projection — *flattening (information loss)*

**Condition**

$$
\mathbf{W} \in \mathbb{R}^{k \times n}, \quad k < n
$$

Often:

$$
\mathbf{W} = 
\begin{bmatrix}
\mathbf{w}^\top_{1} \\
\mathbf{w}^\top_{2} \\
\vdots
\end{bmatrix}
$$

Each row is a direction you keep.  
Extracting less number of features compared to number of input dimensions

**What happens geometrically**
- Some dimensions are **collapsed to zero**
- Multiple points map to the same output
- Non-invertible

**Mental anchor**
> Shadow on a wall — depth is gone forever

**Neural network meaning**
- Dimensionality reduction
- Bottlenecks
- PCA-like behavior
- Compression layers

> Once projected, **no downstream layer can recover what was destroyed**

That is why bottlenecks are powerful and dangerous.

## 3. Embedding — *extrusion (higher dimension)*

**Condition**

$$
\mathbf{W} \in \mathbb{R}^{m \times n}, \quad m > n
$$

Common structure:
- Rows are **learned features**
- Columns spread input into multiple channels 

**What happens geometrically**
- Original space lives inside a higher-D space
- No new information is created
- But **structure becomes linearly separable**

**Mental anchor**
> Lifting a 2D knot into 3D so it can be untangled

**Neural network meaning**
- Feature expansion
- Hidden layers
- Kernel trick (implicitly)

> Embedding $\neq$ adding information  
> Embedding = **making relationships easier to express**

This is *critical* for expressivity

## 4. Stretch / scale — *emphasize directions*

**Condition**

$$
\mathbf{W} = diag(\lambda_{1}, \lambda_{2}, ...)
$$

Or more generally:

$$
\mathbf{W} = \mathbf{U} \mathbf{\Sigma} \mathbf{V}^\top
$$

**What happens geometrically**
- Some axes stretched
- Others shrunk
- Relative importance changes

**Mental anchor**
>Turning volume knobs on different traits

**Neural network meaning**
- Feature weighting
- Activation sensitivity
- Gradient amplification or suppression

> Large singular values = dominant learned concepts  
> Small ones = ignored structure

## 5. Shear / skew — *mix axes asymmetrically*

**Condition**

$$
\mathbf{W} = 
\begin{bmatrix}
1 & k \\
0 & 1
\end{bmatrix}
$$

**What happens geometrically**
- One axis slides relative to another
- Angles distorted
- Parallel lines preserved

**Mental anchor**
> Pushing the top of a box sideways

**Neural network meaning**
- Feature interactions
- Correlated feature mixing
- Non-orthognal representations

This is where **entanglement begins**

## 6. "Learn new axes" — *the core neural insight*

**Condition**

Each neuron computes:

$$
y_{i} = \mathbf{w}^\top_{i} \mathbf{x}
$$

This is **projection onto a learned direction**.

Stacked:

$$
\mathbf{y} = \mathbf{W} \mathbf{x}
$$

**What happens geometrically**
- Each row of $\mathbf{W}$ defines a **concept direction**
- The layer outputs coordinates *in that concept basis*

**Mental anchor**
> The network invents its own coordinate system for meaning

This is the key sentence
> **A neural network layer is a learned basis change + scaling + projection**.

## 7. Why this reframes "learning"

The network is not learning *facts*  
It is learning **what directions in space matter**

Each neuron asks:
> "How much does this input align with *my* idea of something meaningful?"

That's:

$$
\mathbf{w}^\top_{i} \mathbf{x}
$$

## 8. One unifying view

Every linear layer can be decomposed as:

$$
\mathbf{W} = \mathbf{U} \mathbf{\Sigma} \mathbf{V}^\top
$$

Meaning:
1. Rotate into a useful basis (&\mathbf{V}^\top&)
2. Stretch / suppress dimensions ($\mathbf{\Sigma}$)
3. Rotate again ($\mathbf{U}$)
> Every layer = **re-express → emphasize → re-express**

# Final anchor sentence

> Linear algebra is not about numbers.  
It is about **how information is reshaped, compressed, and reinterpreted in space**.
