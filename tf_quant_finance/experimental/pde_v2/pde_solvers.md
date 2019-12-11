<!-- 
After updating, convert this document into pde_solvers.pdf as follows:

pandoc --toc --highlight-style=tango --variable urlcolor=blue -o pde_solvers.pdf pde_solvers.md
-->

---
title: PDE Solvers in Tensorflow Finance
geometry: margin=3cm
---

\renewcommand{\b}{\mathbf}
\renewcommand{\d}{\partial}

# Introduction

The `experimental.pde_v2` package provides ops to compute numerical solutions of
linear parabolic second order partial differential equations.

We currently support equations of the following form:

\begin{equation}
    \frac{\d V}{\d t} +
    \sum\limits_{i,j=1}^N D_{ij}(\b x, t) \frac{\d^2{V}}{\d x_i \d x_j} +
    \sum\limits_{i=1}^N\mu_i(\b x, t) \frac{\d V}{\d x_i} +
     r(\b x, t) V = 0.
\end{equation}

The equation is evolved backwards in time: given $V(\b x, t_f)$, the solver
approximates $(\b x, t_0)$, where $t_0 < t_f$. Support for the forward
time evolution (as well as the Fokker-Planck equation) is coming soon.

The spatial grid (i.e. the grid of $\b x$ vectors) can be arbitrary in
one-dimensional problems ($N = 1$). In multiple dimensions the grid should be
rectangular and uniform in each dimension (the spacing in each dimension can be
different, however).

We support [Robin](https://en.wikipedia.org/wiki/Robin_boundary_condition)
boundary conditions on each edge of the spatial grid:

\begin{equation}
\alpha(\b x_b, t) V(\b x_b, t) + \beta(\b x_b, t)\frac{\d V}{\d \b n}(\b x_b, t)
= \gamma(\b x_b, t), 
\label{boundcond}
\end{equation}

where $\b x_b$ is a point on the boundary, and $\d V/\d \b n$ is the
derivative with respect to the outer normal to the boundary. In particular,
Dirichlet ($\alpha \equiv 1, \beta \equiv 0$) and Neumann
($\alpha \equiv 0, \beta \equiv 1$) boundary conditions are supported.

Below we describe in detail how to use the solver, and then the algorithms it
uses internally.

# Usage guide

## Simplest case: 1D PDE with constant coefficients

Let's start with a one-dimensional PDE with constant coefficients:

\begin{equation}
 \frac{\d V}{\d t} + D \frac{\d^2{V}}{\d x^2} + \mu \frac{\d V}{\d x} + r V = 0.
\end{equation}

Let the domain be $x \in [-1, 1]$, the distribution at the final time be
Gaussian, and the boundary conditions - Dirichlet with zero value on both
boundaries:
\begin{equation}
\begin{split}
&V(x, t_f) = e^{-x^2/2\sigma^2} (2\pi\sigma)^{-1/2},\\
&V(-1, t) = 0,\\
&V(1, t) = 0.
\end{split}
\end{equation}

We're seeking $V(x, t_0).$

Let's prepare the necessary ingredients. First, the spatial grid:

```python
grid = grids.uniform_grid(minimums=[-1],
                          maximums=[1],
                          sizes=[300],
                          dtype=tf.float32)
```

This grid is uniform with 300 points between (and including) $x=-1$ and
$x=1$. The `grids` module provides other types of grids, for example a
log-spaced grid. The `grid` object is a list of `Tensors`. Each element in the
list represents a spatial dimension. In our example `len(grid) = 1` and
`grid[0].shape = (300,)`.

We can also easily make a custom grid out of a numpy array:

```python
grid_np = np.array(...)
grid = [tf.constant(grid_np, dtype=tf.float32)]
```

The next ingredient is the PDE coefficients:

```python
d = 1
mu = 2
r = 3

def second_order_coeff_fn(t, grid):
  return [[d]]

def first_order_coeff_fn(t, grid):
  return [mu]

def zeroth_order_coeff_fn(t, grid):
  return r
```

Note the square brackets - these are required for conformance with
multidimensional case.

Next, the values at the final time $t_f$:

```python
variance = 0.2
xs = grid[0]
final_value_grid = (tf.math.exp(-xs**2 / (2 * variance)) / 
    tf.math.sqrt(2 * np.pi * variance)
```

And finally, define the final time, initial time, and number of time steps:

```python
t_f = 1
t_0 = 0
num_time_steps = 100
```
Just as the spatial grid, the temporal grid can be customized: we can specify
number of time steps, a size of a step, or even a callable that accepts the
current time and returns the size of the next time step.

We now have all the ingredients to solve the PDE:

```python
result_value_grid, final_grid, end_time, steps_performed =
    fd_solvers.step_back(
        start_time=t_f,
        end_time=t_0,
        num_steps=num_time_steps,
        coord_grid=grid,
        values_grid=final_value_grid,
        second_order_coeff_fn=second_order_coeff_fn,
        first_order_coeff_fn=first_order_coeff_fn,
        zeroth_order_coeff_fn=zeroth_order_coeff_fn)
```
The resulting approximation of $V(x, t_0)$ is contained in
`result_value_grid`. The solver also yields the final spatial grid (because the
solver may modify the grid we've provided), the end time (because it may not be
exactly the one we specified, e.g. in case of custom time step callable), and
the number of performed steps.

To visualize the result, we can simply convert it to a numpy array (assuming
we use Tensorflow 2.x or are in the eager mode of Tensorflow 1.x), and apply the
usual tools:

```python
import matplotlib.pyplot as plt

xs = final_grid[0].numpy()
vs = result_value_grid.numpy()
plt.plot(xs, vs)
```
*TODO: insert pictures.*


## Non-constant PDE coefficients: Black-Scholes equation.

Let's now turn to an example of a PDE with non-constant coefficient - the Black-
Scholes equation:

\begin{equation}
 \frac{\d V}{\d t} + \frac12 \sigma^2 S^2 \frac{\d^2{V}}{\d S^2} +
 rS \frac{\d V}{\d S} - r V = 0.
\end{equation}

All we need to change in the previous example is the PDE coefficient callables:

```python
sigma = 1
r = 3

def second_order_coeff_fn(t, grid):
  s = grid[0]
  return [[sigma**2 * s**2 / 2]]

def first_order_coeff_fn(t, grid):
  s = grid[0]
  return [r * s]

def zeroth_order_coeff_fn(t, grid):
  return -r
```

As seen, the coordinates are extracted from the `grid` passed into the
callables, and then can undergo arbitrary Tensorflow transormations. The
returned tensors must be either scalars or have a shape implied by the `grid`
(this is easy to do in 1D, and a bit more tricky in multidimensional case, more
details are below).

That's it, we can now numerically solve the Black-Scholes equation.

## Batching

The solver can work with multiple PDEs in parallel. For example, let's solve
Black-Scholes equation for European options with a batch of payoff values.
European options imply the final condition $V(S, t_f) = (S - K)_+$, where `K`
is the payoff. So with one payoff value we write:

```python
payoff = 0.3
s = grid[0]
final_value_grid = tf.nn.relu(s - payoff)
```

With a batch of `b` payoff values, we need to stack the final value grids, so
that `final_value_grid[i]` is the grid for `i`th payoff value. The simplest way
to do this is by using `tf.meshgrid`:

```python
payoffs = tf.constant([0.1, 0.3, 0.5])
s = grid[0]
payoffs, s = tf.meshgrid(payoffs, s, indexing='ij')
final_value_grid = tf.nn.relu(s - payoffs)
```

`tf.meshgrid` broadcasts the two tensors into a rectangular grid, and then we
can combine them with any algebraic operations.
Make sure that the batch dimension is the first one in the resulting Tensor.

There can be arbitrary number of batching dimensions, which is convenient when
there are multiple parameters:

```python
param1 = tf.constant(...)
param2 = tf.constant(...)
s = grid[0]
param1, param2, s = tf.meshgrid(param1, param2, s, indexing='ij')
final_value_grid = ...  # combine param1, param2 and s
```

After constructing `final_value_grid`, we pass it to `fd_solvers` as usual, and
the `result_value_grid` will contain the batch of solutions.

We may also have different models for each element of the batch, for example:

```python
payoffs = tf.constant([0.1, 0.3, 0.5])
sigmas = tf.constant([1.0, 1.5, 2])
rs = tf.constant([0.0, 1.0, 2.0])

s = grid[0]
payoffs, s = tf.meshgrid(payoffs, s, indexing='ij')
final_value_grid = tf.nn.relu(s - payoffs)

def second_order_coeff_fn(t, grid):
  s = grid[0]
  sigmas_mesh, s_mesh = tf.meshgrid(sigmas, s, indexing="ij")
  return [[sigmas_mesh**2 * s_mesh**2 / 2]]

def first_order_coeff_fn(t, grid):
  s = grid[0]
  rs_mesh, s_mesh = tf.meshgrid(rs, s, indexing="ij")
  return [rs_mesh * s_mesh]

def zeroth_order_coeff_fn(t, grid):
  s = grid[0]
  rs_mesh, s_mesh = tf.meshgrid(rs, s, indexing="ij")
  return -rs_mesh
```

This way we construct three PDEs: `i`-th equation has payoff `payoffs[i]`
and model parameters `sigmas[i]`, `rs[i]`.

In the simplest case, the batch shapes of `final_value_grid` and PDE coefficient
tensors match exactly, like in the last example. The precise requirement is as
follows. The shape of value grid is composed of `batch_shape` and `grid_shape`.
Both are determined from the shape of `final_value_grid`. The dimensonality
`dim` of the PDE, i.e. the the rank of `grid_shape`, is inferred from the `grid`
passed into the solver: `dim = len(grid)` (in all examples so far, `dim = 1`).
`grid_shape` may evolve
with time, so should be determined from the `grid` argument passed into
`second_order_coeff_fn`, `first_order_coeff_fn` and `zeroth_order_coeff_fn`.
Recall that `grid` is a List of 1D Tensors; `grid_shape` is a concatenation of
shapes of these tensors. The requirement is that 
`second_order_coeff_fn(...)[i][j]`,
`first_order_coeff_fn(...)[i]` and `zeroth_order_coeff_fn(...)` must be tensors
whose shape is
[broadcastable](https://docs.scipy.org/doc/numpy/user/basics.broadcasting.html)
to the shape `batch_shape + grid_shape`.

This explains why we can't return just `-rs` from `zeroth_order_coeff_fn` in the
example: the shape of `rs` is `(3,)`, which is not broadcastable to the required
shape `batch_shape + grid_shape = (3, 300)`.

The boundary conditions (see below) can be also batched in a similar way.
The coordinate grid and the temporal grid cannot be batched.

## Boundary conditions

The solver supports boundary conditions in Robin form:

\begin{equation}
\alpha(\b x_b, t) V(\b x_b, t) + \beta(\b x_b, t)\frac{\d V}{\d \b n}(\b x_b, t)
= \gamma(\b x_b, t), 
\end{equation}

where $\b x_b$ is a point on the boundary, $\d V/\d n$ is the derivative
with respect to the outer normal to the boundary. The functions
$\alpha, \beta, \gamma$ can be arbitrary.

The boundary conditions are specified by callables returning
$\alpha, \beta, \gamma$ as Tensors or scalars. For example, a boundary
condition

\begin{equation}
V + 2 \frac{\d V}{\d \b n} = 3 
\end{equation}

is defined as follows:

```python 
def boundary_cond(t, grid):
  return 1, 2, 3
```

Dirichlet and Neumann boundary conditions can be specified by setting
$\alpha = 1, \beta = 0$ and $\alpha = 0, \beta = 1$ explicitly, or by using
utilities found in `boundary_conditions.py`:

```python
@boundary_conditions.dirichlet
def dirichlet_boundary(t, grid):
  return 1  # V = 1

@boundary_conditions.neumann
def neumann_boundary(t, grid):
  return 2  # dV/dn = 2
```

Note that the derivative is taken with respect to the outer normal to the
boundary, not along the coordinate axis. So, for example, the condition
$(\d V/\d x)_{x=x_{min}} = 2$ translates to
$(\d V/\d \b n)_{x=x_{min}} = -2$.

The callables are passed into solver as a list of pairs, where each pair
represents the lower and the upper boundary of a particular spatial dimension.
In 1D this looks as follows:

```python
grid = grids.uniform_grid(minimums=[-1], maximums=[1], ...)

def lower_boundary(t, grid):
  return ...  # boundary condition at x = -1

def upper_boundary(t, grid):
  return ...  # boundary condition at x = 1

result_value_grid, final_grid, end_time, steps_performed =
    fd_solvers.step_back(...,
        boundary_conditions=[(lower_boundary, upper_boundary)])
```

## Multidimensional PDEs; Hull-Heston-White equation.

As a first example of a multi-dimensional PDE, let's consider a 2D PDE with
constant coefficients:

\begin{equation}
\frac{\d V}{\d t} +
D_{xx} \frac{\d^2{V}}{\d x^2} + 2D_{xy} \frac{\d^2{V}}{\d x \d y} +
D_{yy} \frac{\d^2{V}}{\d y^2} +
\mu_x \frac{\d V}{\d x} + \mu_y \frac{\d V}{\d y} + r V = 0
\end{equation}

First, we create a rectangular grid:

```python
x_min, x_max = -1.0, 1.0
y_min, y_max = -2.0, 2.0
x_size, y_size = 200, 300

grid = grids.uniform_grid(minimums=[y_min, x_min],
                          maximums=[y_max, x_max],
                          sizes=[y_size, x_size],
                          dtype=tf.float32)
```

The `grid` object is a list of two Tensors of shapes (300,) and (200,).

Currently, only uniform grids are supported in the multidimensional case.
However, the steps of the grid can be different in each dimension.

The PDE coefficient callables look as follows:

```python
d_xx, d_xy, d_yy = 1, 0.2, 0.5
mu_x, mu_y = 1, 0.5
r = 3

def second_order_coeff_fn(t, grid):
  return [[d_yy, d_xy], [d_xy, d_xx]]

def first_order_coeff_fn(t, grid):
  return [mu_y, mu_x]

def zeroth_order_coeff_fn(t, grid):
  return r
```

The matrix returned by `second_order_coeff_fn` can be a list of lists like
above, a Numpy array or a Tensorflow tensor - the requirement is that
`second_order_coeff_fn(...)[i][j]` is defined for `i <= j < dim` and represents
the corresponding PDE coefficient. The matrix is assumed symmetrical, and
elements with `i > j` are never accessed. Therefore, they can be anything,
so the following is also acceptable[^nones]:

```python
def second_order_coeff_fn(t, grid):
  return [[d_yy, d_xy], [None, d_yy]]
```

[^nones]: `None` can also be used to indicate that certain PDE terms are absent.
  For example, when $\mu_x = 0$, we can return `[mu_y, None]` from
  `first_order_coeff_fn`. If both $\mu_x, \mu_y = 0$, we can return
  `[None, None]` or simply pass `first_order_coeff_fn=None` into the solver.

The final values grid can be constructed, as usual, with the help of
`tf.meshgrid`. For example,

\begin{equation}
V(x, y, t_f) =  (2\pi\sigma)^{-1} e^{-\frac{x^2 + y^2}{2\sigma^2}}
\end{equation}

translates into[^tf_tensordot]


```python
sigma = 0.1
ys, xs = grid
y_mesh, x_mesh = tf.meshgrid(ys, xs, indexing="ij")
final_value_grid = (tf.math.exp(-(x_mesh**2 + y_mesh**2) / (2 * sigma))
    / (2 * np.pi * sigma))
```

[^tf_tensordot]: With multiplicatively separable functions, we can avoid
  creating `x_mesh, y_mesh` with the help of `tf.tensordot`:
    ```python
    sigma = 0.1
    ys, xs = grid
    final_value_grid = (tf.tensordot(tf.math.exp(-ys**2 / (2 * sigma)),
                                     tf.math.exp(-xs**2 / (2 * sigma)), 
                                     [[], []]) / (2 * np.pi * sigma))
    ```

Finally, call `fd_solvers.step_back` as usual:

```python
t_f = 1
t_0 = 0
num_time_steps = 100

result_value_grid, final_grid, end_time, steps_performed =
    fd_solvers.step_back(
        start_time=t_f,
        end_time=t_0,
        num_steps=num_time_steps,
        coord_grid=grid,
        values_grid=final_value_grid,
        second_order_coeff_fn=second_order_coeff_fn,
        first_order_coeff_fn=first_order_coeff_fn,
        zeroth_order_coeff_fn=zeroth_order_coeff_fn)
```

One way to visualize the result is by creating a heatmap:

```python
plt.imshow(result_value_grid.numpy(), 
           extent=[x_min, x_max, y_min, y_max],
           cmap='hot')
plt.show()
```

The boundary of the domain is rectangular, and each side can has its
boundary condition (by default - Dirichlet with zero value). For example, let's
"heat up" the right boundary and set zero flux across y-boundaries:

```python
@boundary_conditions.dirichlet
def boundary_x_max(t, grid):
  return 1.0

@boundary_conditions.dirichlet
def boundary_x_min(t, grid):
  return 0.0

@boundary_conditions.neumann
def boundary_y_max(t, grid):
  return 0.0

@boundary_conditions.neumann
def boundary_y_min(t, grid):
  return 0.0

result_value_grid, final_grid, end_time, steps_performed = fd_solvers.step_back(
    ...
     boundary_conditions=[(boundary_y_min, boundary_y_max),
                          (boundary_x_min, boundary_x_max)])
```

The boundary conditions can be inhomogeneous. E.g. a "heat source" near
$x = x_{max}, y = 0$ may look like this:

```python
sigma = 0.2

@boundary_conditions.dirichlet
def boundary_x_max(t, grid):
  ys = grid[0]
  return tf.exp(-ys**2 / (2 * sigma**2))
```

The general requirement for the shape of the boundary condition tensors is that
they must be either scalars or tensors of shape
`batch_shape + grid_shape'`, where `grid_shape'` is `grid_shape`
excluding the axis orthogonal to the boundary. For example, in 3D the value
grid shape is `batch_shape + (z_size, y_size, x_size)`, and the boundary tensors
on the planes $y=y_{min}$ and $y=y_{max}$ should be either scalars or
have shape `batch_shape + (z_size, x_size)`.

As a more elaborate example, let's translate into code the Heston-Hull-White
PDE, as defined in [1]:

\begin{equation}
\begin{split}
&\frac{\d\varphi}{\d t} + \frac12 s^2 v \frac{\d^2\varphi}{\d s^2} +
\frac12 \sigma_1^2 v \frac{\d^2\varphi}{\d v^2} +
\frac12 \sigma_2^2 v \frac{\d^2\varphi}{\d r^2} + \\
&\rho_{12} \sigma_1 s v \frac{\d^2\varphi}{\d s \d v} +
\rho_{13} \sigma_2 s \sqrt{v} \frac{\d^2\varphi}{\d s \d r} +
\rho_{23} \sigma_1 \sigma_2 \sqrt{v} \frac{\d^2\varphi}{\d v \d r} + \\
&rs \frac{\d\varphi}{ds} + \kappa(\eta - v) \frac{\d\varphi}{dv} +
a(bt-r)\frac{\d\varphi}{dr} - ru = 0
\end{split}
\end{equation}

with boundary conditions

\begin{equation}
\begin{split}
&\varphi(0, v, r, t) = 0, \\
&\frac{\d\varphi}{\d s}(s_{max}, v, r, t) = 1, \\
&\varphi(s, 0, r, t) = 0, \\
&\varphi(s, v_{max}, r, t) = s, \\
&\frac{\d\varphi}{\d r}(s, v, r_{min}, t) = 0, \\
&\frac{\d\varphi}{\d r}(s, v, r_{max}, t) = 0. \\
\end{split}
\end{equation}


```python
def second_order_coeff_fn(t, grid):
  s, v, r = tf.meshgrid(grid[0], grid[1], grid[2], indexing='ij')
  coeff_ss = s**2 * v / 2
  coeff_vv = sigma1**2 * v / 2
  coeff_rr = sigma2**2 * v / 2
  coeff_sv = rho12 * sigma1 * s * v / 2
  coeff_sr = rho13 * sigma2 * s * tf.sqrt(v) / 2
  coeff_vr = rho23 * sigma1 * sigma2 * tf.sqrt(v) / 2
  return [[coeff_ss, coeff_sv, coeff_sr],
          [None, coeff_vv, coeff_vr],
          [None, None, coeff_rr]]

def first_order_coeff_fn(t, grid):
  # Note how we avoid high-dimensional meshgrids and use tf.expand_dims to
  # make the shapes broadcastable to (s_size, v_size, r_size).
  s, v, r = grid
  s_mesh, r_mesh = tf.meshgrid(s, r, indexing='ij')
  coeff_s = r_mesh * s_mesh  # shape is (s_size, r_size)
  coeff_s = tf.expand_dims(coeff_s, axis=1)  # shape is (s_size, 1, r_size)

  coeff_v = kappa * (eta - v)  # shape is (v_size,)
  coeff_v = tf.expand_dims(coeff_v, axis=-1)  # shape is (v_size, 1)

  coeff_r = a * (b * t - r)  # shape is (r_size,)
  return [coeff_s, coeff_v, coeff_r]

def zeroth_order_coeff_fn(t, grid):
  return -r

@boundary_conditions.dirichlet
def boundary_s_min(t, grid):
  return 0.0

@boundary_conditions.neumann
def boundary_s_max(t, grid):
  return 1.0

@boundary_conditions.dirichlet
def boundary_v_min(t, grid):
  return 0.0

@boundary_conditions.dirichlet
def boundary_v_max(t, grid):
  s_mesh, r_mesh = tf.meshgrid(grid[0], grid[2], indexing='ij')
  return s_mesh  # shape is (s_size, r_size)

@boundary_conditions.neumann
def boundary_r_min(t, grid):
  return 0.0

@boundary_conditions.neumann
def boundary_r_max(t, grid):
  return 0.0
```

## Customizing the time marching scheme

The solver allows specifying a time marching scheme. Time marching schemes
define how a single time step is performed (after spatial discretization of a
PDE), and they differ in numerical stability, accuracy, and performance.

To use, for example, the explicit scheme, which is less accurate and stable but
much faster than the default (Crank-Nicolson) scheme, we write:

```python
result_value_grid, final_grid, end_time, steps_performed = 
    fd_solvers.step_back(...
        one_step_fn=fd_backward_schemes.explicit.explicit_step)
```

Currently the following schemes are supported for 1D:

1. Explicit,
2. Implicit,
3. Crank-Nicolson,
4. Weighted explicit-implicit,
5. Extrapolation scheme, [3]
6. Crank-Nicolson with oscillation damping (see discussion below).

For multidimensional problems we currently have:

1. Douglas ADI [1, 4].

By default, the Crank-Nicolson with oscillation damping is used for 1D problems
and Douglas ADI with $\theta = 0.5$ - for multidimensional problems.

See below a detailed discussions of these schemes.

# Implementation

The bird's eye view of a finite-difference PDE solving algorithm is

```
for time_step in time_steps:       (1)
   do_spatial_discretization()     (2)
   apply_time_marching_scheme()    (3)
```

The spatial discretization converts a PDE for $V(\b x, t)$ into a linear
system of equations for $\b V(t)$, which is a vector consisting of
$V(\b x_i, t),$ where $\b x_i$ are grid points. The resulting system of
equations has the form

\begin{equation}
\frac{d\b V}{dt} = \hat L(t)\b V + \b b(t),\label{space_discretized}
\end{equation}

where $\b b$ is a vector, and $\hat L$ is a sparse matrix describing the
influence of adjacent grid points on each other. $\hat L(t)$ and $\b b(t)$
are defined by both PDE coefficients and the boundary conditions, evaluated at
time $t$.

The job of a time marching scheme is to approximately solve
Eq.\eqref{space_discretized} for $\b V(t + \delta t)$ given $\b V(t)$
($\delta t$ may be positive or negative).

All three lines in the above pseudocode represent the routines that are mostly
independent from each other. Therefore, the solver is modularized accordingly:

- `fd_solvers.py` implement looping over time steps, i.e. line (1) in the 
pseudocode. The routines accept (and provide reasonable defaults for)
`one_step_fn` argument, which is a callable performing both (2) and (3).

- `parabolic_equation_stepper.py` and `multidim_parabolic_equation_stepper.py`
perform space discretization for one-dimensional and multidimensional parabolic
PDEs, respectively. They accept a `time_marching_scheme` callable for performing
(3).

- `crank_nicolson.py`, `explicit.py`, `douglas_adi.py` and other modules
implement specific time marching schemes. Each of these modules has a
`<scheme_name>_scheme` function for the scheme itself, and a
`<scheme_name>_step` function. The latter is a convenience function which
combines the given scheme and the appropriate space discretizer; it can be
passed directly to `fd_solvers`.

This setup is flexible, allowing for easy customization of every component.
For example, it's straightforward to implement a new time marching scheme and
plug it in.

Below are the implementation notes for space discretizers and the time marching
schemes.

## Spatial discretization

The spatial derivatives are approximated to the second order of accuracy, and
taking into account that the grid may be non-uniform. Consider first the 1D case,
and denote $x_0$ the given point in the grid, and $\Delta_+, \Delta_-$ the
distances to adjacent grid points.
The Taylor's expansion of a function $f(x)$ with smooth second derivative
gives
\begin{eqnarray}
f(x_0 + \Delta_+) &=& f(x_0) +  f'(x_0) \Delta_{+}
    + \frac{1}{2}f''(x_0) (\Delta_{+})^2 + \mathcal{O}((\Delta_{+})^3),
    \label{taylorplus}\\
f(x_0 - \Delta_-) &=& f(x_0) - f'(x_0) \Delta_{-}
    + \frac{1}{2}f''(x_0) (\Delta_{-})^2 + \mathcal{O}((\Delta_{-})^3).
    \label{taylorminus}
\end{eqnarray}

Solving this system of equations for $f'(x_0)$ and $f''(x_0)$ yields
\begin{equation}
f'(x_0) \approx \frac{\Delta_-}{\Delta_+ + \Delta_-} f'_+ +
    \frac{\Delta_+}{\Delta_+ + \Delta_-} f'_-
\label{deriv_approx_central}
\end{equation}

and
\begin{equation}
f''(x_0) \approx 2\frac{f'_+ - f'_-}{\Delta_+ + \Delta_-},
\end{equation}

where $f'_{\pm} = \pm(f(x_0 + \Delta_\pm) - f(x_0))/ \Delta_{\pm}$ are the
forward and backward estimates of the first derivative.

Thus, space discretization a 1D homogeneous parabolic PDE yields
Eq.\eqref{space_discretized} with $\b b = 0$, and a tridiagonal matrix
$\hat L$.

In multidimensional case the discretization is done similarly, but we
additionally need to take care of mixed second derivatives. Since only uniform
grids are currently supported in multidimensional case, we apply the usual
approximation of mixed derivatives:

\begin{equation}
\frac{\d^2 f}{\d x \d y}(x_0, y_0) \approx \frac{
f(x_+, y_+) - f(x_-, y_+) - f(x_+, y_-) + f(x_-, y_-)} {4\Delta_x\Delta_y},
\end{equation}

where $x_\pm = x_0 \pm \Delta_x, y_\pm = y_0 \pm \Delta_y$ are adjacent grid
points.

Considering that $\b V$ in Eq.\eqref{space_discretized} is a vector, i.e. a
flattened multidimensional value grid, the matrix $\hat L$ is now banded,
with contributions coming from adjacent points in every dimension.

### Boundary conditions

The most general form of boundary conditions we support is the Robin boundary
conditions, see Eq. \eqref{boundcond}. When discretizing Eq. \eqref{boundcond},
we approximate the derivative to the second order of accuracy, just like we did
with the PDE terms. The central approximation, Eq. \eqref{deriv_approx_central},
is however not applicable, because it uses the values at adjacent points on both
sides. Instead, we express the boundary derivative via the two closest points on
one side. Consider the lower boundary $x_0$, and the two
closest grid points $x_1 = x_0 + \Delta_0$, and $x_2 = x_1 + \Delta_1$:
\begin{eqnarray}
f(x_1) &\approx& f(x_0) +  f'(x_0) \Delta_0
    + \frac12 f''(x_0) \Delta_0^2,\\
f(x_2) &\approx& f(x_0) + f'(x_0) (\Delta_0 + \Delta_1)
    + \frac12 f''(x_0) (\Delta_0 + \Delta_1)^2\\
\end{eqnarray}

Eliminating $f''(x_0)$ from this system, we obtain
\begin{equation}
f'(x_0) \approx \frac
{
  (\Delta_0 + \Delta_1)^2 f(x_1) - \Delta_0^2 f(x_2)
  - \Delta_0 ( 2\Delta_0 + \Delta_1 ) f(x_0)
}{
 \Delta_0\Delta_1 ( \Delta_0 + \Delta_1 )
}
\end{equation}

(see e.g. [2], §2.3; only uniform grids are considered there though).

Similarly, we express the derivative at the upper boundary $x_n$ via the two
closest points on the left of it: $x_{n-1} = x_n - \Delta_{n-1}$ and
$x_{n-2} = x_{n-1} - \Delta_{n-2}$. The expression differs essentially only in
the sign:
\begin{equation}
f'(x_n) \approx -\frac
{
  (\Delta_{n-1} + \Delta_{n-2})^2 f(x_{n-1}) - \Delta_{n-1}^2 f(x_{n-2})
  - \Delta_{n-1} ( 2\Delta_{n-1} + \Delta_{n-2} ) f(x_n)
}{
 \Delta_{n-1}\Delta_{n-2} ( \Delta_{n-1} + \Delta_{n-2} )
}
\end{equation}

Substituting these two equations into Eq. \eqref{boundcond} (taking into
account that $\d V/\d \b n = - \d V/\d x$ on the lower boundary, and
$\d V/\d \b n = \d V/\d x$ on the upper boundary), we get the following
discretized versions of it:
\begin{eqnarray}
V_0 = \xi_1 V_1 + \xi_2 V_2 + \eta,\label{lower_bound_discr}\\
V_n = \bar\xi_1 V_{n-1} + \bar\xi_2 V_{n-2} + \bar\eta,\label{upper_bound_discr}
\end{eqnarray}

where
\begin{eqnarray}
&&\xi_1 = \beta (\Delta_0 + \Delta_1)^2 / \kappa, \\
&&\xi_2 = -\beta \Delta_0^2 / \kappa, \\
&&\eta = \gamma \Delta_0 \Delta_1 (\Delta_0 + \Delta_1) / \kappa, \\
&&\kappa = \alpha \Delta_0 \Delta_1 (\Delta_0 + \Delta_1) +
   \beta \Delta_1 (2\Delta_0 + \Delta_1).
\end{eqnarray}

The expressions for $\bar\xi_1, \bar\xi_2$ and $\bar\eta$ are exactly the
same, except $\Delta_{1, 2}$ is replaced by $\Delta_{n-1, n-2}$, and of
course $\alpha, \beta$ and $\gamma$ come from the upper boundary condition.

The evolution of the values on the inner part of the grid, $V_0, \ldots
V_{n-1}$ is defined by a tridiagonal matrix, as discussed in the previous
section:
\begin{equation}
\frac {d V_i}{dt} = L_{i, i-1} V_{i-1} + L_{i, i} V_{i} + L_{i, i+1} V_{i+1},
\qquad i = 1\ldots N-1
\end{equation}

Substituting Eqs. \eqref{lower_bound_discr}, \eqref{upper_bound_discr}, we obtain
for the inner part $\b V_{inner} = [V_1, ... V_{n-1}]^T$:

\begin{equation}
\frac {d \b V_{inner}}{dt} = \hat{\tilde L} \b V_{inner} + \b b,
\label {dVdt_with_boundary}
\end{equation}

where 
\begin{eqnarray} 
&&\tilde L_{11} = L_{11} + \xi_1 L_{01}, \qquad \tilde L_{12} =
L_{12} + \xi_2 L_{01},\label{bound_corr_first} \\ &&\tilde L_{n-1, n-1} =
L_{n-1,n-1} + \bar\xi_1 L_{n,n-1}, \qquad \tilde L_{n-1, n-2} = L_{n-1,n-2} +
\bar\xi_2 L_{n,n-1}, \\ &&\tilde L_{ij} = L_{ij} \qquad i=2\ldots n-2,\\ &&b_1 =
L_{01} \eta, \qquad b_{n-1} = L_{n, n-1}\bar \eta, \\ &&b_i = 0 \qquad i=2\ldots
n-2.\label{bound_corr_last} 
\end{eqnarray}

Note that in case of Dirichlet conditions ($\alpha = 1, \beta = 0$) this
simplifies greatly: $\xi_{1,2} = 0$ (so there are no corrections to
$\hat L$), and $\eta = \gamma$.

Thus to take into account the boundary conditions we

*   apply the corrections to the time evolution equation given by Eqs.
    \eqref{bound_corr_first}-\eqref{bound_corr_last},
*   apply the chosen time marching scheme to find the "inner" part of the values
    vector $\b V_{inner}(t_{i+1})$ from Eq.
    \eqref{dVdt_with_boundary} given $\b V_{inner}(t_i)$,
*   and finally, find $V_0$ and $V_n$ from Eqs. \eqref{lower_bound_discr},
    \eqref{upper_bound_discr} at time $t + \delta t$, and append them to
    $\b V_{inner}(t_{i+1})$ to get the resulting vector $\b V(t_{i+1})$.

This approach generalizes straightforwardly to the multidimensional case:
$\b V_{inner}$ would be the value grid with all the boundaries "trimmed", and
Eqs. \eqref{bound_corr_first}-\eqref{bound_corr_last} would similarly apply to the
items next to the boundary.

## Time marching schemes

Time marching schemes are algorithms for numerically solving the equation

\begin{equation}
\frac{d\b V}{dt} = \hat L(t)\b V + \b b(t),\label{space_discretized2}
\end{equation}

for $\b V(t + \delta t)$ given $\b V(t),$ with a small $\delta t$.

Note that in case of constant parameters $\hat L$ and $\b b$ this equation
has an exact solution involving the matrix exponent of $\hat L$. Calculating
the matrix exponent is however infeasible in practice, primarily due to memory
constraints. The matrix $\hat L$ is never explicitly constructed in the first
place. Recall that in 1D the matrix $\hat L$ is tridiagonal, and in
multidimensional case it is banded. Only the non-zero diagonals are constructed,
and the time marching schemes make use of the sparseness of $\hat L.$

### Explicit Scheme

The simplest scheme is the explicit scheme:

\begin{equation}
\frac{\b V_1 - \b V_0}{\delta t} = \hat L_0 \b V_0 + \b b_0.
\end{equation}

Here and below we use the notation $X_\alpha = X(t + \alpha \delta t)$ where
$X$ is any function of time and $\alpha$ is a number between 0 and 1.

From there we obtain

\begin{equation}
\b V_1 = (1 + \delta t\hat L_0) \b V_0 + \delta t \b b_0.
\end{equation}

The calculation boils down to multiplying a tridiagonal matrix by a vector.
Tensorflow has the
[tf.linalg.tridiagonal_matmul](https://www.tensorflow.org/api_docs/python/tf/linalg/tridiagonal_matmul)
Op, which can efficiently parallelize the multiplication on the GPU.

Unfortunately, the explicit scheme is applicable only with small time steps.
For a well-defined PDE, the matrix $\delta t \hat L$ has negative eigenvalues
(This statement includes PDEs solved both in forward and backward directions,
i.e. $\delta t$ can be both positive and negative). When an eigenvalue
of $1 + \delta t \hat L_0$ becomes less than $-1$, the contribution to
$\b V$ of the corresponding eigenvector grows exponentially with time. This
does not happen to the exact solution, so the error of the approximate solution
grows exponentially, i.e. the method is unstable. The stability requirements
are $\delta t \ll \Delta x^2 / D$ and $\delta t \ll \Delta x / \mu,$ where
$D$ and $\mu$ are the typical values of second and first order coefficients
of a PDE. This can constrain $\delta t$ to unacceptably small values.

### Implicit Scheme

The implicit scheme approximates the right hand side with its value at
$t + \delta t:$
\begin{equation}
\frac{\b V_1 - \b V_0}{\delta t} = \hat L_1 \b V_1 + \b b_1,
\end{equation}

yielding
\begin{equation}
\b V_1 = (1 - \delta t\hat L_0)^{-1} (\b V_0 + \delta t \b b_1).
\end{equation}

This takes care of the stability problem, because the eigenvalues of
$(1 - \delta t\hat L_0)^{-1}$ are between 0 and 1. However, we now need to
solve a tridiagonal system of equation.

Tensorflow has the
[tf.linalg.tridiagonal_solve](https://www.tensorflow.org/api_docs/python/tf/linalg/tridiagonal_solve)
Op for this purpose. On GPU, it uses
[gtsv](https://docs.nvidia.com/cuda/cusparse/index.html#gtsv) routines from
Nvidia's CUDA Toolkit. These routines employ Parallel Cyclic Reduction algorithm
(see e.g. [5]) which enables certain level of parallelization.

### Weighted Implicit-Explicit Scheme and Crank-Nicolson Scheme

Weighted Implicit-Explicit Scheme is a combination of implicit and explicit
schemes:

\begin{equation}
\frac{\b V_1 - \b V_0}{\delta t} = \theta(\hat L_0 \b V_0 + \b b_0) +
(1-\theta)(\hat L_1 \b V_1 + \b b_1),
\end{equation}

where $\theta$ is a number between 0 and 1.

This yields
\begin{equation}
\b V_1 = (1 - (1-\theta)\delta t\hat L_0)^{-1} [
(1+\theta \delta_t \hat L_1)\b V_0 + \theta\delta t \b b_0 +
(1-\theta)\delta t \b b_1].
\end{equation}

The special case of $\theta = 1/2$ is the Crank-Nicolson (CN) scheme:
\begin{equation}
\b V_1 = (1 - \delta t\hat L_0 / 2)^{-1} [
(1+\Delta_t \hat L_1 / 2)\b V_0 + \delta t (\b b_0 + \b b_1) / 2].
\label{cn}
\end{equation}

CN scheme is stable (the eigenvalues are between -1 and 1), and,
unlike schemes with any other value of $\theta$, is second-order accurate in
$\delta t.$

One can verify (by comparing the Taylor expansions) that replacing
$\hat L_{0,1}, \b b_{0,1}$ with $\hat L_{1/2}, \b b_{1/2}$ in the right-hand
side of Eq.\eqref{space_discretized} retains the second order accuracy, while
saving some computation. Thus the final expression is
\begin{equation}
\b V_1 = (1 - \delta t\hat L_{1/2} / 2)^{-1} [
(1+\delta t \hat L_{1/2} / 2)\b V_0 + \delta t \b b_{1 / 2}].
\end{equation}


### Oscillations in Crank-Nicolson Scheme and Extrapolation Scheme

In financial applications, the initial or final data is often not smooth.
For example, the call option payoff has a discontinuity in the first
derivative at the strike. This can cause a severe drop in the accuracy of the
Crank-Nicolson scheme: it results in oscillations around the points of
discontinuity.

The reason of this effect is Crank-Nicolson scheme poorly approximating the
evolution of high-wavenumber components of the initial/final condition
function.

For the sake of discussion, assume $\delta t > 0$, $\b b(t) = 0$ and
$\hat L(t) = -\hat A$ (note that $\hat A$ has positive eigenvalues in this
case). The exact solution of Eq. \eqref{space_discretized2} is then

\begin{equation}
\mathbf V(t + \delta t) = \exp(-\hat A \delta t) \mathbf V(t).
\end{equation}

Various time marching schemes can be viewed as approximations to the exponent.
Crank-Nicolson scheme corresponds to the approximation

\begin{equation}
\exp(-y) \approx \frac{1-y/2}{1+y/2}.
\end{equation}

Here $y=\lambda \delta t$ and $\lambda$ is an
eigenvalue of $\hat A$. This approximation, while being second order accurate
for $y \ll 1$, is clearly inaccurate for $y \gtrsim 1$, and because it is
negative there, the oscillations arise.

There are a few approaches to this problem found in the literature.

1) Replacing CN scheme with schemes based on Richardson extrapolation, i.e.
taking a linear combination of results of making time steps of different sizes,
with coefficients chosen to achieve desired accuracy. The simplest scheme
corresponds to the following approximation [3]:
\begin{equation}
\exp(-y) \approx \frac 2{(1 + y/2)^2} - \frac 1 {1 + y}.
\end{equation}
This means we are making two implicit half-steps, one implicit full step, and
combine the results. This approximation is second order accurate for small
$y$, just like CN, but is a more reasonable approximation of the exponent for
large $y$. The cost of this improvement is having to solve three tridiagonal
systems per step, compared to one system and one tridiagonal multiplication in
CN scheme. The implementation of this scheme can be found in `extrapolation.py`.
Higher orders of accuracy can be also achieved in this fashion. For example,
combining two CN half-steps and CN full step with coefficients 4/3 and -1/3
yields a third-order accurate approximation [6].

2) Subdiving the first time step into multiple smaller steps [7]. The idea is
that after the first step the irregularities in the initial
conditions are already smoothened to a large enough degree that CN scheme can be
applied to all further steps. But this smoothening itself needs to be modelled
more accurately. By reducing $\delta t$ we can ensure $y \lesssim 1$, for
all components of initial/final condition function.

3) Subdividing the first step into two fully implicit half-steps ("Rannacher
marching"), [8]. This is more computationally efficient than the two other
approaches. However, the fully implicit steps, while indeed damping the
oscillations, cause significant errors for low-wavenumber components, because of
its first-order accuracy on small wavenumbers.

The optimal approach seems to be a combination of all three ideas: apply the
extrapolation scheme to the first time step, or a few first steps, then proceed
with CN scheme. This is mentioned in the end of [8]. Such approach adds
minimal computational burden: only a finite number of extra tridiagonal systems
to solve. It ensures damping of oscillations, i.e. a reasonable treatment of
high-wavenumber components, while maintaining second-order accuracy of
low-wavenumber components.

The high-wavenumber components are dampened by approximately $(\delta t\max_i
(\lambda_i))^{-n_e},$ where $n_e$ is number of extrapolation steps taken. One
can approximate the maximum eigenvalue, and tune $n_e$ to achieve desired
level of oscillation damping.

The implementation of this can be found in
`oscillation_damped_crank_nicolson.py`.


### Multiple spatial dimensions, ADI

In case of multiple spatial dimensions, the matrix $\hat L$ is banded, making
it much more difficult to do the matrix inversion in the Crank-Nicolson scheme.

The common workaround is the ADI (alternating direction explicit) method.
The time step is split into several substeps, and each substep treats only one
dimension explicitly. There is a number of time marching schemes based on this
idea. An overview of them can be found, for example, on pages 6-9 of [1]. Here
we summarize the simplest of them, the Douglas scheme, in our notations:

\begin{eqnarray}
\b Y_0 &=& (1 + \hat L_0 \delta t) \b V_0 + \delta t \b b_0, \\
\b Y_j &=& \b Y_{j-1} + \theta \delta t (\hat L^{(j)}_1 \b Y_j -
\hat L^{(j)}_0 \b V_0 + \b b^{(j)}_1 - \b b^{(j)}_0), \qquad j = 1\ldots dim, \\
\b V_1 &=& \b Y_{dim}.
\end{eqnarray}

Here $\hat L^{(j)}$ contains contributions to $$L$ coming from derivatives
with respect to j-th dimension. Each $\hat L^{(j)}$ thus have only three
nonzero diagonals. The contribution of the zeroth-order term
is split evenly between $\hat L^{(j)}$. The contributions of mixed
second-order terms are not included in any $\hat L^{(j)}$, and thus take part
only in the first substep, which lacks any matrix inversion.
$\b b^{(j)}$ are contributions from boundary conditions along the j-th axis
(boundary conditions are assumed to be the only source of $\b b$).
$\theta$ is a positive number.

The scheme is second-order accurate if there are no mixed derivative terms and
$\theta = 1/2,$ and first-order accurate otherwise. It is stable if
$\theta \geq 1/2$.

The implementation can be found in `douglas_adi.py`. The main computational
burden is solving large batches of tridiagonal systems plus transposing the
value grid Tensor on each substep. To understand why the latter is necessary,
recall that in implementation, $\b V$ is not a vector, but a Tensor of shape
`batch_shape + grid_shape`. When making the substep which treats j-th dimension
implicitly, we should treat all the other dimensions as batch dimensions.

For example, consider a 3D case where the shape is
`batch_shape + (z_size, y_size, x_size)`. When making the substep with respect
that treats `y` implicitly, we transpose the value grid to the shape
`batch_shape + (z_size, x_size, y_size)`, and send it to
`tf.linalg.tridiagonal_solve`. From the point of view of the latter, the batch
shape is `batch_shape + (z_size, x_size)`, exactly as we need. After that we
transpose the result back to the original shape.

# References

[1] [Tinne Haentjens, Karek J. in't Hout. ADI finite difference schemes for
the Heston-Hull-White PDE](https://arxiv.org/abs/1111.4087)

[2] [S. Mazumder. Numerical Methods for Partial Differential Equations. ISBN
9780128498941. 2015.](https://www.sciencedirect.com/book/9780128498941/numerical-methods-for-partial-differential-equations)

[3] [D. Lawson, J & Ll Morris, J. The Extrapolation of First Order Methods
for Parabolic Partial Differential Equations. I. 1978
SIAM Journal on Numerical Analysis. 15. 1212-1224.](https://epubs.siam.org/doi/abs/10.1137/0715082)

[4] [Douglas Jr., Jim (1962), "Alternating direction methods for three space
variables", Numerische Mathematik, 4 (1): 41-63.](https://dl.acm.org/citation.cfm?id=2722576)

[5] [D. Göddeke, R. Strzodka, "Cyclic reduction tridiagonal solvers on GPUs
applied to mixed precision multigrid", IEEE Transactions on Parallel and
Distributed System, vol. 22, pp. 22-32, Jan. 2011.](https://ieeexplore.ieee.org/abstract/document/5445081)

[6] [A. R. Gourlay and J. Ll. Morris. The Extrapolation of First Order Methods
for Parabolic Partial Differential Equations, II. SIAM Journal on Numerical
Analysis Vol. 17, No. 5 (Oct., 1980), pp. 641-655.](https://www.jstor.org/stable/2156665?seq=1)

[7] [D.Britz, O.Østerby, J.Strutwolf. Damping of Crank–Nicolson error
oscillations. Computational Biology and Chemistry, Vol. 27, Iss. 3, July 2003,
Pages 253-263.](https://www.sciencedirect.com/science/article/pii/S009784850200075X)

[8] [Giles, Michael & Carter, Rebecca. (2005). Convergence analysis of
Crank-Nicolson and Rannacher time-marching. J. Comput. Finance. 9.](https://www.researchgate.net/publication/228524629_Convergence_analysis_of_Crank-Nicolson_and_Rannacher_time-marching)