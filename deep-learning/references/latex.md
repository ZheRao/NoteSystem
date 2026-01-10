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
```

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
> $\mathbf{w}^\top \mathbf{x}$

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
> $\mathbf{W}^\top \mathbf{W} = \mathbf{I}$

# 4. Norms, distances, magnitudes

```latex
\|\mathbf{x}\|
\|\mathbf{x}\|_2
\|\mathbf{x}\|_1
\|\mathbf{x}\|_\infty
```
> $$
> \|\mathbf{x}\|,  
> \|\mathbf{x}\|_2,  
> \|\mathbf{x}\|_1,  
> \|\mathbf{x}\|_\infty
> $$

**Distance**

```latex
\|\mathbf{x} - \mathbf{y}\|
```
> $\|\mathbf{x} - \mathbf{y}\|$

# 5. Functions, mappings, and geometry

**Functions**:

```latex
f(\mathbf{x})
```
> $f(\mathbf{x})$

**Linear map**:

```latex
f(\mathbf{x}) = \mathbf{W}\mathbf{x}
```
> $f(\mathbf{x}) = \mathbf{W}\mathbf{x}$

**Affine map**:

```latex
f(\mathbf{x}) = \mathbf{W}\mathbf{x} + \mathbf{b}
```
> $f(\mathbf{x}) = \mathbf{W}\mathbf{x} + \mathbf{b} $

**Activation**

```latex
\sigma(\mathbf{W}\mathbf{x} + \mathbf{b})
```
> $\sigma(\mathbf{W}\mathbf{x} + \mathbf{b})$

# 6. Derivatives, gradients, Jacobians (deep learning core)

**Scalar derivative**

```latex
\frac{dL}{dx}
```
> $$ \frac{dL}{dx} $$

**Partial derivative**

```latex
\frac{\partial L}{\partial x}
```
> $$ \frac{\partial L}{\partial x} $$

**Gradient (vector)**

```latex
\nabla_{\mathbf{x}} L
```
> $$ \nabla_{\mathbf{x}} L $$

**Jacobian**

```latex
\frac{\partial \mathbf{y}}{\partial \mathbf{x}}
```
> $$ \frac{\partial \mathbf{y}}{\partial \mathbf{x}} $$

**Chain rule (backprop essence)**

```latex
\frac{\partial L}{\partial \mathbf{x}}
=
\frac{\partial L}{\partial \mathbf{y}}
\frac{\partial \mathbf{y}}{\partial \mathbf{x}}
```
> $$
> \frac{\partial L}{\partial \mathbf{x}} =
> \frac{\partial L}{\partial \mathbf{y}}
> \frac{\partial \mathbf{y}}{\partial \mathbf{x}}
> $$

# 7. Expectations, probability, extropy (later but inevitable)

**Expectation**:

```latex
\mathbb{E}[X]
\mathbb{E}_{x \sim p(x)}[f(x)]
```

> $\mathbb{E}[X]$  
> $\mathbb{E}_{x \sim p(x)}[f(x)]$

**Variance**

```latex
\mathrm{Var}(X)
```
> $\mathrm{Var}(X)$

**Entropy**

```latex
H(p) = - \sum_x p(x) \log p(x)
```
> $H(p) = - \sum_x p(x) \log p(x)$

# 8. Common ML symbols

```latex
\theta      % parameters
\mathcal{L} % loss
\arg\min
\arg\max
```
> $\theta$, $\mathcal{L}$, $\arg\min$, $\arg\max$

# 9. Clean multi-line equations (Readbility)

Use `aligned`:

```latex
$$
\begin{aligned}
\mathbf{z} &= \mathbf{W}\mathbf{x} + \mathbf{b} \\
\mathbf{y} &= \sigma(\mathbf{z})
\end{aligned}
$$
```
> $$
> \begin{aligned}
> \mathbf{z} &= \mathbf{W}\mathbf{x} + \mathbf{b} \\
> \mathbf{y} &= \sigma(\mathbf{z})
> \end{aligned}
> $$

# 10. Inner products, projections (geometry intuition)

**inner product**:

```latex
\langle \mathbf{w}, \mathbf{x} \rangle
```
> $\langle \mathbf{w}$, $\mathbf{x} \rangle$

**Projection**:

```latex
\mathrm{proj}_{\mathbf{w}}(\mathbf{x})
=
\frac{\mathbf{w}^\top \mathbf{x}}{\mathbf{w}^\top \mathbf{w}} \mathbf{w}
```
> $$
> \mathrm{proj}_{\mathbf{w}}(\mathbf{x}) =
> \frac{\mathbf{w}^\top \mathbf{x}}{\mathbf{w}^\top \mathbf{w}} \mathbf{w}
> $$
