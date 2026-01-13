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


## 2. Projection 
