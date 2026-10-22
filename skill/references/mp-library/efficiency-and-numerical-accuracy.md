# Efficiency and Numerical Accuracy

Derived from: mp-efficiency-considerations.md, mp-numerical-accuracy.md

Source: `mp-efficiency-considerations.md` (`https://mp.ampl.com/modeling-efficiency.html`),
`mp-numerical-accuracy.md` (`https://mp.ampl.com/modeling-numerics.html`).

MP's expanded expression support (`expressions-and-modeling-guide.md`) lets you write a
model the way you naturally think about it and let the interface handle the conversion to
solver-ready form. That conversion doesn't always produce the *fastest-solving* or most
*numerically stable* formulation, though — a handful of formulation choices reliably help
both. Future MP versions may automate more of these reformulations; for now, applying them
by hand is worthwhile on models where solve time or accuracy is a concern. For a general
treatment beyond these MP-specific points, see E. Klotz and A.M. Newman (2013), "Practical
guidelines for solving difficult mixed integer linear programs," *Surveys in Operations
Research and Management Science* 18(1-2), 18-32.

## Efficiency

### Simplify the logic first

Before reformulating, check whether the model can simply be smaller: are all the
variables, constraints, and logical conditions actually needed? Beyond that, a logically
equivalent restatement of a constraint can reduce the number of logical conditions MP has
to reformulate, which generally speeds up solving:

```ampl
# Original: one binary + implication per (p,i,j) combination
sum {(i,j) in ARCS}
   if exists {p in PRODUCTS} Flow[p,i,j] > 0 then fix_cost[i,j];

# Equivalent, but the solver works with one linear expression per (i,j) instead
sum {(i,j) in ARCS}
   if sum {p in PRODUCTS} Flow[p,i,j] > 0 then fix_cost[i,j];
```

For a given `i,j`, "some `Flow[p,i,j]` is positive" is exactly equivalent to "the sum of
all `Flow[p,i,j]` is positive" — the rewritten form takes advantage of the solver's
native handling of linear expressions instead of instantiating a separate binary variable
and implication constraint per product `p`.

### Create common subexpressions

Where two constraints (or a constraint and the objective) can be restated to share the
literal same subexpression, MP's conversion can reuse the reformulation rather than
building it twice. For example, if a constraint already implies that
`sum {p in PRODUCTS} Flow[p,i,j]`, when positive, must be at least `min_ship`:

```ampl
subject to Shipment_Limits {(i,j) in ARCS}:
   sum {p in PRODUCTS} Flow[p,i,j] = 0 or
   min_ship <= sum {p in PRODUCTS} Flow[p,i,j] <= capacity[i,j];
```

then an objective term written as `> 0` can be safely rewritten as `>= min_ship`:

```ampl
minimize TotalCost:
   ... + sum {(i,j) in ARCS}
      if sum {p in PRODUCTS} Flow[p,i,j] >= min_ship then fix_cost[i,j];
```

Now `sum {p in PRODUCTS} Flow[p,i,j] >= min_ship` appears verbatim in both the constraint
and the objective, simplifying the model MP has to build.

### Bound subexpressions explicitly

Tightening variable bounds is one of the most reliable ways to improve solver performance
— and where a subexpression's bounds are implied by its constituent variables' bounds but
not obvious to the solver, making that bound explicit via an auxiliary variable helps.
Consider the Goldstein-Price test function:

```ampl
var x {1..2} <= 2, >= -2;

minimize Goldstein_Price:
   (1 + (x[1] + x[2] + 1)^2
      * (19 - 14*x[1] + 3*x[1]^2 - 14*x[2] + 6*x[1]*x[2] + 3*x[2]^2))
 * (30 + (2*x[1] - 3*x[2])^2
      * (18 - 32*x[1] + 12*x[1]^2 + 48*x[2] - 36*x[1]*x[2] + 27*x[2]^2));
```

The bounds on `x[1]`, `x[2]` imply bounds on the repeated subexpressions
`(x[1] + x[2] + 1)^2` (∈ [0, 25]) and `(2*x[1] - 3*x[2])^2` (∈ [0, 100]) — but the solver
only sees that if it's told explicitly. Defining auxiliary variables for these
subexpressions, with their bounds stated directly, communicates that information:

```ampl
var t1 >= 0, <= 25;   subj to t1def: t1 = (x[1] + x[2] + 1)^2;
var t2 >= 0, <= 100;  subj to t2def: t2 = (2*x[1] - 3*x[2])^2;

minimize Goldstein_Price:
   (1 + t1 * (19 - 14*x[1] + 3*x[1]^2 - 14*x[2] + 6*x[1]*x[2] + 3*x[2]^2))
 * (30 + t2 * (18 - 32*x[1] + 12*x[1]^2 + 48*x[2] - 36*x[1]*x[2] + 27*x[2]^2));
```

This is observed to substantially improve Gurobi's performance on this model.

## Numerical accuracy

Solvers work in finite precision, so numerical stability is a real concern independent of
formulation correctness. MP's own guidance narrows to three recurring, concrete points.

### Keep coefficients close to ±1

Rescale data so constraint-matrix and objective coefficients stay close to ±1 wherever
practical. This is a general MIP/LP numerical-stability practice, not MP-specific, but it
matters more once MP is auto-generating additional constraints during reformulation.

### Keep bounds tight — especially for "big-M" constraints

MP simplifies relational operators (`==>`, indicator-style conditions) into "indicator"
constraints where possible. Solvers with native indicator-constraint support usually
handle these in a numerically stable way. Where a solver lacks that support, MP has to
linearize the indicator using a "big-M" constant instead — for example,

```ampl
b==0 ==> x<=0;
```

may become

```ampl
x <= upper_bound(x) * b;
```

with the big-M taken from `x`'s upper bound. This linearization also gets used for the
MIP's continuous relaxation, so it affects solving speed as well as just the final answer.
**Big-M constraints require finite bounds on every participating variable** — and those
bounds should be as tight as possible, ideally within ±10⁴. In any case, for numerical
stability they should never exceed the reciprocal of the integrality tolerance
(`inttol`). A default big-M can be set with `cvt:bigM` (use cautiously — a default that's
too loose defeats the purpose, but the right value is model-dependent).

Sometimes a different formulation makes automatic bound deduction possible where the
original one couldn't. The `==>`/`else` form below can't easily deduce bounds on `DL`:

```ampl
var DL {l in Lset, s in Sset};
subject to setDL {l in Lset, s in Sset}:
   L[l,s] > K[l,s] ==> DL[l,s] = L[l,s] else DL[l,s] = 0;
```

Reformulated with `if-then`, the same logic lets MP deduce bounds automatically:

```ampl
subject to setDL {l in Lset, s in Sset}:
   DL[l,s] = if L[l,s] > K[l,s] then L[l,s];
```

### Piecewise-linear functions and stability

Piecewise-linear expressions — whether written directly with `<<...>>` or arising from an
approximation of another nonlinear function (`expressions-and-modeling-guide.md`, "General
nonlinear functions") — are numerically stable when the solver handles them with a native
PL algorithm. When they're linearized instead, bound the argument and result variables
within ±10⁴ (tighter is better). For approximated nonlinear functions specifically, MP
imposes hard bounds of up to ±10⁶ by default if none are given. Where stability is still a
problem after tightening bounds, further options include lowering the integrality
tolerance, Gurobi's `numfocus`/`intfocus` options, and disabling the solver's own presolve
— tuning measures, applied case by case rather than as a first resort.

## See also

- `expressions-and-modeling-guide.md` — the operators (`<<...>>`, `==>`, general
  nonlinear functions) whose reformulation these bounds and tolerances directly affect
- `tools-troubleshooting-and-reformulation.md` — `acc:` acceptance options, exporting the
  solver's model to see the actual big-M/indicator forms MP generated, and general
  troubleshooting steps when a model solves slowly or inaccurately
- `../modeling-patterns/piecewise-linear-programs.md` — the `<<...>>` notation itself
- `../modeling-patterns/nonlinear-programs.md` — NLP-specific pitfalls beyond
  MP-reformulation numerics (local optima, starting-point sensitivity)
- `feature-and-solver-options-guide.md` — `cvt:bigM` and other MIP/numerical-focus options
  in the broader options-mechanics context
