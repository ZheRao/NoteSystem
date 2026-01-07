# 0. How LaTeX works inside `.md`

Most Markdown renders (GitHub, MkDocs, Obsidian, Notion, Jupyter) support **MathJax**.
- Inline math → `$ ... $`
- Block math → `$$ ... $$`

```md
A vector $x \in \mathbb{R}^n$ is transformed by a matrix $W$.

$$
y = Wx
$$
```
> A vector $x \in \mathbb{R}^n$ is transformed by a matrix $W$.
> $$
> y = Wx 
> $$

# 1. Scalars, vectors, matrices

**Scalars**

```latex
a, b, \alpha, \lambda
```
> $a$, $b$, $\alpha$, $\lambda$


**Vectors**

Recommended for ML notes: **bold**

```latex
\mathbf{x}, \mathbf{w}, \mathbf{y}
``

> $\mathbf{x}$, $\mathbf{w}$, $\mathbf{y}$

Alternative

```latex
\vec{x}
```
> $\vec{x}$

**Matrices (upper case bold)**

```latex
\mathbf{W}, \mathbf{A}, \mathbf{\Sigma}
```
> $\mathbf{W}$, $\mathbf{A}$, $\mathbf{\Sigma}$

**Sets and spaces**

```latex
\mathbf{W} \in \mathbb{R}^{m \times n}, \quad \mathbf{x} \in \mathbb{R}^n
```
> $\mathbf{W} \in \mathbb{R}^{m \times n}, \quad \mathbf{x} \in \mathbb{R}^n$

# 2. Matrix-vector multiplication

```latex
\mathbf{y} = \mathbf{W}\mathbf{x}
```
> $\mathbf{y} = \mathbf{W}\mathbf{x}$

**Component-wise**:

```latex
y_i = \sum_{j=1}^{n} W_{ij} x_j
```
> $y_i = \sum_{j=1}^{n} W_{ij} x_j$

**Dot product**:

```latex
\mathbf{w}^\top \mathbf{x}
```
> $ \mathbf{w}^\top \mathbf{x} $

# 3. Transpose, inverse, identity

```latex
\mathbf{W}^\top
\mathbf{W}^{-1}
\mathbf{I}
```
> $\mathbf{W}^\top$  
> $\mathbf{W}^{-1}$  
> $\mathbf{I}$

**Orthogonality**:

```latex
\mathbf{W}^\top \mathbf{W} = \mathbf{I}
```
> $ \mathbf{W}^\top \mathbf{W} = \mathbf{I} $

# 4. Norms, distances, magnitudes

```latex
\|\mathbf{x}\|
\|\mathbf{x}\|_2
\|\mathbf{x}\|_1
\|\mathbf{x}\|_\infty
```
> $\|\mathbf{x}\|$  
> $\|\mathbf{x}\|_2$  
> $\|\mathbf{x}\|_1$  
> $\|\mathbf{x}\|_\infty$

**Distance**

```latex
\|\mathbf{x} - \mathbf{y}\|
```
> $ \|\mathbf{x} - \mathbf{y}\| $

# 5. Functions, mappings, and geometry

**Functions**:

```latex
f(\mathbf{x})
```
> $ f(\mathbf{x}) $

**Linear map**:

```latex
f(\mathbf{x}) = \mathbf{W}\mathbf{x}
```
> $f(\mathbf{x}) = \mathbf{W}\mathbf{x}$

**Affine map**:

```latex
f(\mathbf{x}) = \mathbf{W}\mathbf{x} + \mathbf{b}
```
