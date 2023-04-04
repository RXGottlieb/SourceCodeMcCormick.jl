# SourceCodeMcCormick.jl

This package is an experimental approach to use source-code transformation to apply McCormick relaxations
to symbolic functions for use in deterministic global optimization. While packages like `McCormick.jl` [1]
take set-valued McCormick objects and utilize McCormick relaxation rules to overload standard math operations,
`SourceCodeMcCormick.jl` (SCMC) aims to interpret symbolic expressions, apply generalized McCormick rules,
create source code that computes the McCormick relaxations and natural interval extension of the input,
and compile the source code into functions that return pointwise values of the natural interval extension
and convex/concave relaxations. This functionality is designed to be used for both algebraic and dynamic
(in development) systems.

## Algebraic Systems

For a given algebraic equation or system of equations, `SCMC` is designed to provide symbolic transformations 
that represent the lower/upper bounds and convex/concave relaxations of the provided equation(s). Most notably, 
`SCMC` uses this symbolic transformation to generate "evaluation functions" which, for a given expression, 
return the natural interval extension and convex/concave relaxations of an expression. E.g.:

```
using SourceCodeMcCormick, Symbolics

@variables x, y
expr = exp(x/y) - (x*y^2)/(y+1)
expr_lo_eval, expr_hi_eval, expr_cv_eval, expr_cc_eval, order = all_evaluators(expr)
```

Here, the outputs marked `_eval` are the evaluation functions for the lower bound (`lo`), upper
bound (`hi`), convex underestimator (`cv`), and concave overestimator (`cc`) of the symbolic
expression given by `expr`. The inputs to each of these functions are described by the `order` 
vector, which in this case is `[x_cc, x_cv, x_hi, x_lo, y_cc, y_cv, y_hi, y_lo]`, representing
the concave/convex relaxations and interval bounds of the variables `x` and `y`. E.g., if being
used in a branch-and-bound (B&B) scheme, the interval bounds for each variable will be the lower and
upper bounds of the B&B node for that variable, and the convex/concave relaxations will take the
value where the relaxation of the original expression is desired.

One benefit of using a source code transformation approach such as this over a multiple dispatch
approach like `McCormick.jl` is speed. When McCormick relaxations of functions are evaluated using
`McCormick.jl`, there is overhead associated with finding the correct functions to call for each
overloaded math operation. The functions generated by `SCMC`, however, eliminate this overhead by
creating static functions with the correct McCormick rules already applied. While the `McCormick.jl`
approach is flexible in quickly evaluating any new expression you provide, in the `SCMC` approach,
one expression is selected up front, and relaxations and interval extension values are calculated
for that expression quickly. For example:

```
using BenchmarkTools, McCormick

xMC = MC{2, NS}(2.5, Interval(-1.0, 4.0), 1)
yMC = MC{2, NS}(1.5, Interval(0.5, 3.0), 2)

@btime exp(xMC/yMC) - (xMC*yMC^2)/(yMC+1)
# 497.382 ns (7 allocations: 560 bytes)

@btime expr_cv_eval(2.5, 2.5, 4.0, -1.0, 1.5, 1.5, 3.0, 0.5)
# 184.964 ns (1 allocation: 16 bytes)
```

Note that this is not an entirely fair comparison because `McCormick.jl`, by using the `MC` type and
multiple dispatch, simultaneously calculates all of the following: natural interval extensions,
convex and concave relaxations, and corresponding subgradients. 

Another benefit of the `SCMC` approach is its compatibility with `CUDA.jl` [2]: `SCMC` functions are
broadcastable over `CuArray`s. Depending on the GPU, number of evaluations, and complexity of the
function, this can dramatically decrease the time to compute the numerical values of interval
extensions and relaxations. E.g.:

```
using CUDA

# Using McCormick.jl
xMC_array = MC{2,NS}.(rand(10000), Interval.(zeros(10000), ones(10000)), ones(Int, 10000))
yMC_array = MC{2,NS}.(rand(10000), Interval.(zeros(10000), ones(10000)), ones(Int, 10000).*2)

@btime @. exp(xMC_array/yMC_array) - (xMC_array*yMC_array^2)/(yMC_array+1)
# 2.365 ms (18 allocations: 703.81 KiB)


# Using SourceCodeMcCormick.jl, broadcast using CPU
xcc = rand(10000)
xcv = copy(xcc)
xhi = ones(10000)
xlo = zeros(10000)

ycc = rand(10000)
ycv = copy(xcv)
yhi = ones(10000)
ylo = zeros(10000)

@btime expr_cv_eval.(xcc, xcv, xhi, xlo, ycc, ycv, yhi, ylo);
# 1.366 ms (20 allocations: 78.84 KiB)


# Using SourceCodeMcCormick.jl and CUDA.jl, broadcast using GPU
xcc_GPU = CuArray(xcc)
xcv_GPU = CuArray(xcv)
xhi_GPU = CuArray(xhi)
xlo_GPU = CuArray(xlo)
ycc_GPU = CuArray(ycc)
ycv_GPU = CuArray(ycv)
yhi_GPU = CuArray(yhi)
ylo_GPU = CuArray(ylo)

@btime CUDA.@sync expr_cv_eval.(xcc_GPU, xcv_GPU, xhi_GPU, xlo_GPU, ycc_GPU, ycv_GPU, yhi_GPU, ylo_GPU);
# 29.800 μs (52 allocations: 3.88 KiB)
```


## Dynamic Systems

(In development) For dynamic systems, `SCMC` assumes a differential inequalities approach where the 
relaxations of derivatives are calculated in advance and the resulting (larger) differential equation 
system, with explicit definitions of the relaxations of derivatives, can be solved. For algebraic 
systems, the main product of this package is the broadcastable evaluation functions. For dynamic
systems, this package follows the same idea as in algebraic systems but stops at the symbolic 
representations of relaxations. This functionality is designed to work with a `ModelingToolkit`-type 
`ODESystem` with factorable equations [3]--`SCMC` will take such a system and return a new `ODESystem` 
with expanded equations to provide interval extensions and (if desired) McCormick relaxations. E.g.:

```
using SourceCodeMcCormick, ModelingToolkit
@parameters p[1:2] t
@variables x[1:2](t)
D = Differential(t)

tspan = (0.0, 35.0)
x0 = [1.0; 0.0]
x_dict = Dict(x[i] .=> x0[i] for i in 1:2)
p_start = [0.020; 0.025]
p_dict = Dict(p[i] .=> p_start[i] for i in 1:2)

eqns = [D(x[1]) ~ p[1]+x[1],
        D(x[2]) ~ p[2]+x[2]]

@named syst = ODESystem(eqns, t, x, p, defaults=merge(x_dict, p_dict))
new_syst = apply_transform(McCormickIntervalTransform(), syst)
```

This takes the original ODE system (`syst`) with equations:
```
Differential(t)(x[1](t)) ~ x[1](t) + p[1]
Differential(t)(x[2](t)) ~ x[2](t) + p[2]
```

and generates a new ODE system (`new_syst`) with equations:
```
Differential(t)(x_1_lo(t)) ~ p_1_lo + x_1_lo(t)
Differential(t)(x_1_hi(t)) ~ p_1_hi + x_1_hi(t)
Differential(t)(x_1_cv(t)) ~ p_1_cv + x_1_cv(t)
Differential(t)(x_1_cc(t)) ~ p_1_cc + x_1_cc(t)
Differential(t)(x_2_lo(t)) ~ p_2_lo + x_2_lo(t)
Differential(t)(x_2_hi(t)) ~ p_2_hi + x_2_hi(t)
Differential(t)(x_2_cv(t)) ~ p_2_cv + x_2_cv(t)
Differential(t)(x_2_cc(t)) ~ p_2_cc + x_2_cc(t)
```

where `x_lo < x_cv < x < x_cc < x_hi`. Only addition is shown in this example, as other operations
can appear very expansive, but the same operations available for algebraic systems are available
for dynamic systems as well. As with the algebraic evaluation functions, equations created by
`SourceCodeMcCormick` are GPU-ready--multiple trajectories of the resulting ODE system at 
different points and with different state/parameter bounds can be solved simultaneously using
an `EnsembleProblem` in the SciML ecosystem, and GPU hardware can be applied for these solves
using `DiffEqGPU.jl`.

## Limitations

`SCMC` has several limitations, some of which are described here. Ongoing research effort seeks
to address several of these.
- SCMC does not calculate subgradients, which are used in the lower bounding routines of many 
global optimizers
- Complicated expressions may cause significant compilation time. This can be manually avoided
by combining results together in a user-defined function
- `SCMC` is currently compatible with elementary arithmetic operations +, -, *, /, and the
univariate intrinsic functions ^2 and exp. More diverse functions will be added in the future
- Functions created with `SCMC` may only accept 32 CUDA arrays as inputs, so functions with more
than 8 unique variables will need to be split/factored by the user to be accommodated
- Due to the large number of floating point calculations required to calculate McCormick-based
relaxations, it is highly recommended to use double-precision floating point numbers, including
for operations on a GPU. Since most GPUs are designed for single-precision floating point operation,
forcing double-precision will often result in a significant performance hit. GPUs designed for
scientific computing, with a higher proportion of double-precision-capable cores, are recommended
for optimal performance with `SCMC`.
- Due to the high branching factor of McCormick-based relaxations and the possibility of warp
divergence, there will likely be a performance gap between optimizations with variables covering
positive-only domains and variables with mixed domains. Additionally, more complicated expressions
where the structure of a McCormick relaxation changes more frequently with respect to the bounds
on its domain will likely perform worse than problems where the structure of the relaxation is
more consistent.

## References

1. M.E. Wilhelm, R.X. Gottlieb, and M.D. Stuber, PSORLab/McCormick.jl (2020), URL
https://github.com/PSORLab/McCormick.jl.
2. T. Besard, C. Foket, and B. De Sutter, Effective extensible programming: Unleashing Julia
on GPUs, IEEE Transactions on Parallel and Distributed Systems (2018).
3. Y. Ma, S. Gowda, R. Anantharaman, C. Laughman, V. Shah, C. Rackauckas, ModelingToolkit: 
A composable graph transformation system for equation-based modeling. arXiv preprint 
arXiv:2103.05244, 2021. doi: 10.48550/ARXIV.2103.05244.