# MP Expressions and Modeling Guide

Derived from: mp-expressions-supported.md

Source: `mp-expressions-supported.md` (`https://mp.ampl.com/modeling-expressions.html`).

MP-based solver interfaces work with existing AMPL syntax but accept it in more general
contexts and across a wider range of solvers than earlier ASL-based drivers did. Two kinds
of entities recur through the syntax summaries below:

- **expr** — any expression that evaluates to a number; may contain variables, and may be
  built from ordinary arithmetic as well as the operators covered here.
- **constr** — a constraint that evaluates to true or false; built from the familiar
  relational operators (`>(=)`, `<(=)`, `=`) as well as operators like `or` and `alldiff`
  that produce constraints directly.

Operators combine freely — an `expr`-valued or `constr`-valued result from one operator can
feed into another, letting AMPL build up arbitrarily complex expression trees:

```ampl
x<=0 or (y!=2  ==>
         x<=-5 else
               max((x+1)*x*z, y, y-z)<=3 and exp(y)<=12);
```

AMPL represents such combinations as expression trees and hands them to the MP-based
interface, which processes them however the target solver requires.

Indexing follows AMPL's usual `{ indexing } expr` form (AMPL book §5.5 and Chapter 6), or
a parenthesized, comma-separated `( expr-list )`, whose entries may themselves be `expr`,
`{indexing} expr`, or a nested `{indexing} ( expr-list )`. `max` accepts both forms:

```ampl
max {n in NODE} weight[t,n] * Use[n]
max (cost["BRO"], cost["CAU"], cost["BRU"])
max ({f in FOOD} cost[f], 10.0)
```

**Non-closed feasible regions.** Because MP accepts general operators like strict
inequalities inside constraints, it's possible to write a model whose feasible region
isn't topologically closed (e.g. `x > 5` with `x` minimized has no minimum). Where
necessary, MP closes the region with a small tolerance (`x >= 5 + eps`); the tolerance is
controlled by option `cvt:mip:eps`. Reformulations MP itself performs are configured
through MP options (`acc:...`, `cvt:...` — see `tools-troubleshooting-and-reformulation.md`);
reformulations AMPL performs before handing off to the solver are configured through AMPL
options.

## Conditional operators

- `if constr then expr1 [else expr2]` — *expr-valued.* Takes `expr1` when `constr` is
  true, `expr2` (or 0) when false. If `constr` contains no variables, AMPL can resolve the
  value before solving; otherwise the whole expression goes to the solver.
- `constr1 ==> constr2 [else constr3]` — *constr-valued.* Satisfied when `constr1` is true
  and `constr2` is true, or when `constr1` is false (and `constr3` holds, if present).
- `constr2 <== constr1` — the reverse-arrow form of the same implication.
- `constr1 <==> constr2` — *constr-valued.* Satisfied when `constr1` and `constr2` agree
  (both true or both false).

```ampl
subject to Balance {p in PROD, t in TIME}:
   Make[p,t] + (if t = 0 then inv0[p] else Inv[p,t-1])
      = Sell[p,t] + Inv[p,t];

subject to Least_Use {j in SCHEDS}:
   Use[j] = 1 ==> Work[j] >= least_assign else Work[j] = 0;
```

Read `constr1 ==> constr2` as "`constr1` implies `constr2`." When `constr1` has the form
*binary-var* = 0 or *binary-var* = 1, this is an "indicator" constraint some solvers
handle natively; otherwise it's transformed into relational-operator constraints.

## Logical operators

- `constr1 or constr2`, `constr1 and constr2`, `not constr` — the usual Boolean
  connectives, *constr*-valued.
- `exists {indexing} constr`, `forall {indexing} constr` — the iterated forms of `or` and
  `and`: satisfied when at least one / all of the indexed `constr` operands hold.

Where possible these transform into Gurobi's native AND/OR "general constraints";
otherwise into simpler relational-operator constraints.

```ampl
subj to No_Conflict {i1 in JOBS, i2 in JOBS: ord(i1) < ord(i2)}:
   Start[i2] >= Start[i1] + t_offset[i1,i2]  or
   Start[i1] >= Start[i2] + t_offset[i2,i1];

subj to HostNever {j in BOATS}:
   isH[j] = 1 ==> forall {t in TIMES} H[j,t] = j;
```

## Piecewise-linear expressions

- `abs(expr)` — `expr` if ≥ 0, else `-expr`.
- `min {indexing} expr` / `min (expr-list)`, `max {indexing} expr` / `max (expr-list)` —
  smallest/largest value among the operands.
- `<< breakpoint-list; slope-list >> var` — a piecewise-linear function of a single
  variable; full notation covered in AMPL book Chapter 17.

`abs`/`min`/`max` transform into Gurobi's native general constraints where possible,
otherwise into relational-operator constraints, linearized when every operand is linear:

```ampl
minimize Max_Cost:
   max {i in PEOPLE} sum {j in PROJECTS} cost[i,j] * Assign[i,j];
```

For the general `<<breakpoints;slopes>>` form, **set AMPL option `pl_linearize` to 0 for
MP solvers.** With `pl_linearize=0`, the piecewise-linear expression reaches the solver
interface as an expression tree, and MP transforms it to the solver's native
piecewise-linear handling (Gurobi, COPT) or linearizes it for solvers without native
support (HiGHS). At AMPL's default `pl_linearize=1`, AMPL itself linearizes the function
before the solver interface ever sees it — continuously in convex/concave cases, with
added binary variables otherwise. When a piecewise-linear function ends up linearized
(either path), bound the argument and result variables tightly (at most `[-1e4, 1e4]` is
recommended) for numerical stability — see `efficiency-and-numerical-accuracy.md`.

```ampl
minimize Total_Cost:
   sum {i in ORIG, j in DEST}
      <<{p in 1..npiece[i,j]-1} limit[i,j,p];
        {p in 1..npiece[i,j]} rate[i,j,p]>> Trans[i,j];
```

**This section extends the treatment of piecewise-linear functions in
`../modeling-patterns/piecewise-linear-programs.md`** — the `<<...>>` notation and
detection rules described there are unchanged, but MP-based solvers add native solver
support as an alternative to AMPL's own LP/MIP linearization, gated by the
`pl_linearize=0` recommendation above.

## Counting operators

- `count {indexing} constr` — *expr-valued:* number of indexing-set members for which
  `constr` holds.
- `atleast k {indexing} constr`, `atmost k {indexing} constr`, `exactly k {indexing}
  constr` — *constr-valued* shorthands for common `count` comparisons (`k` must be a
  nonnegative-integer-valued constant expression).
- `numberof expr in (expr-list)` — *expr-valued:* how many entries of `expr-list` equal
  `expr`.

```ampl
subject to Min_Serve {i in ORIG}:
    atleast minserve {j in DEST} (Ship[i,j] >= minload);

subj to CapacityOfMachine {k in MACHINES}:
    numberof k in ({j in JOBS} MachineForJob[j]) <= cap[k];
```

## Relational and comparison operators, and `alldiff`

- `expr1 > expr2`, `>=`, `<`, `<=`, `==`/`=`, `!=` — the ordinary relational operators,
  *constr*-valued. Where possible (e.g. integer-valued operands, or an `if ... > 0 then
  ...` fixed-cost pattern) strict operators transform into `>=`/`<=`; otherwise a small
  tolerance is introduced, as with the non-closed-region case above. Numerical accuracy
  implications are covered in `efficiency-and-numerical-accuracy.md`.
- `alldiff {indexing} expr` / `alldiff (expr-list)` — *constr*-valued: every operand takes
  a distinct value. A concise alternative to writing `!=` between every pair. No MP-based
  solver currently supports `alldiff` natively; MP always reformulates it.

```ampl
subject to Different_Colors {(c1,c2) in Neighbors}:
   Color[c1] != Color[c2];

subject to OnePersonPerPosition:
   alldiff {i in 1..nPeople} Pos[i];
```

## Complementarity operator

- `constr1 complements constr2` — satisfied when both hold and at least one holds with
  equality; each side must be a single inequality (or the trivial equality case).
- `expr complements constr` / `constr complements expr` — satisfied when `constr` (a
  double inequality `lb <= expr2 <= ub`) holds, and `expr`'s sign pins which bound is
  active: positive `expr` forces the lower bound to equality, negative forces the upper
  bound.

```ampl
subject to Pri_Compl {i in PROD}:
   Price[i] >= 0 complements
      sum {j in ACT} io[i,j] * Level[j] >= demzero[i] - demrate[i] * Price[i];
```

`complements` expresses complementary-slackness/equilibrium relationships directly —
see the full worked derivation, satisfaction rules, and suffix mechanics (`.Lsuf`/`.Rsuf`)
in `../modeling-patterns/complementarity-problems.md`; the AMPL book's Chapter 19 covers
it in full. Knitro handles complementarity constraints natively; for other MP-based
solvers the interface converts `complements` into logical or nonlinear-function
equivalents, controlled by options `cvt:compl` and `cvt:compl:eps`.

## Set membership operator

- `var var-name in set-expr;` — restricts a variable's domain to the (numeric) members of
  `set-expr`. Valid in any `var` definition without an `=` phrase.

**Work in progress (MP library):** this feature is currently under rework upstream. Until
further notice, prefer disjunctive constraints (`||`) over `in set-expr`/`interval[...]`
domain restrictions, especially when the variable also participates in logical expressions:

```ampl
# avoid:
var grade in 0..2 union {5} union interval[8.1,9] union interval[10,13.7];

# prefer:
var grade >= 0 <= 13.7;
s.t. GradeDomain:
    grade <= 0 || grade == 1 || grade == 2 || grade == 5
        || 8.1 <= grade <= 9 || grade >= 10;
```

AMPL rewrites `in set-expr` to an equivalent definition without `in` before the solver ever
sees it — typically introducing auxiliary binary variables and constraints. For a finite
`set-expr`, AMPL also defines suffixes `.sos`/`.sosref`, letting the solver interface
recognize the construct and, where the solver supports it, implement it as a
special-ordered-set-of-type-1 (SOS1) rather than binaries. `set-expr` may also include
continuous intervals via `interval[expr1,expr2]`, making the domain infinite:

```ampl
var Buy {f in FOODS} in {0,10,30,45,55};
var Ship {(i,j) in ARCS} in {0} union interval[min_ship,capacity[i,j]];
```

## Quadratic and power operators

- `expr1 * expr2`, `expr1 / expr2` — multiplication, division.
- `expr1 ^ expr2` — power. For constant positive-integer `expr2`, the operator goes to
  the solver natively if supported, else decomposes into quadratic constraints if the
  solver supports those, else falls back to piecewise-linear approximation (see "General
  nonlinear functions" below).

For `linear * linear` and `linear^2` forms, MP multiplies out the product so individual
quadratic-term coefficients can be extracted and passed to a solver with native quadratic
support; otherwise the terms are linearized where possible (e.g. one operand is binary) or
approximated. For linear solvers this multiply-out/linearize path is the default; for
**convex MIQP solvers**, force linearization instead of native quadratic handling with
`cvt:quadobj=0` / `cvt:quadcon=0`. Piecewise linearization of quadratics also lets convex
MIP solvers handle nonconvex QP/nonlinear models they couldn't otherwise accept.

Other combinations reduce to auxiliary variables plus quadratic expressions/equalities:

```
(x-1)^3  ⇒  (x-1) * y  with added constraint  y = (x-1)^2
x * max {j in 1..n} y[j]  ⇒  x * z  with added constraint  z = max {j in 1..n} y[j]
```

## Conic optimization

Mosek, Gurobi, COPT, SCIP, CPLEX, and Xpress can handle conic constraints with tailored
algorithms (general nonlinear solvers accept the same syntax but without a specialized
conic algorithm). Worked examples: AMPL Colab (`https://colab.ampl.com/tags/conic.html`).

**Second-order cone programming (SOCP).** SOCP constraints (quadratic cones) are
recognized from their algebraic form by major commercial MIP solvers; MP provides
additional conversion into solver-specific conic forms for some representations.

```ampl
# Standard SOC (x[0] >= 0):
x[0] >= sqrt(x[1]^2 + ... + x[n]^2);

# Rotated SOC (x[0], x[1] >= 0):
2*x[0]*x[1] >= x[2]^2 + ... + x[n]^2;
```

Mosek cannot mix SOCP and general quadratic constraints in the same model (errors with
`MSK_RES_ERR_MIXED_CONIC_AND_NL`) — use a standard SOCP form throughout, or tune options
`cvt:socp`/`cvt:socp2qp`.

**Exponential cones.** Mosek natively handles these; MP recognizes and passes them via
Mosek's API:

```ampl
s.t. T1: 1 + b*w >= exp( q1 );
s.t. T2: -1 + w +10*q2   <= -5 * q2 * exp( q1 / (q2*5) );
```

Mosek also cannot mix conic and general quadratic constraints — use `cvt:expcones=0` to
force an alternative handling (piecewise-linear approximation, for Mosek) if needed.

To see which constraints actually reached the solver as conic vs. quadratic, export the
solver's model — see "Export the solver model" in
`tools-troubleshooting-and-reformulation.md`.

## General nonlinear functions

**Supported functions:** `log`, `log10`, `exp`; `sin`, `cos`, `tan`, `asin`, `acos`,
`atan`; `sinh`, `cosh`, `tanh`, `asinh`, `acosh`, `atanh`; and `^` (power), whose handling
depends on which operand is a constant: for two variable operands it goes natively if
supported, else reformulates as `exp(expr2 * log(expr1))`; for constant positive base,
native or piecewise-linear approximation; for constant positive-integer exponent, native,
quadratic decomposition, or piecewise-linear approximation, in that preference order.
Disable native handling with `acc:pow=0`, `acc:expa=0`, or `acc:powconstexp=0` respectively.

**Recognized from the model (native in Gurobi 13):**

- `signpow(x, p) := sign(x) |x|^p` (p > 1) — recognized from forms like
  `abs(x+x*y+7)^0.5*(x+x*y+7)` (→ `signpow(x+x*y+7, 1.5)`) or `sqrt(z^2)^2.3*z` (→
  `signpow(z, 3.3)`, single-variable argument only). Disable with `cvt:pre:signpow=0`.
  Prefer `sqrt` over `abs` for this pattern — `abs` is only eliminated (and the pattern
  recognized) when its argument's deduced bounds have a definite sign; `sqrt` always
  converts, though it's less efficient for solvers without native `signpow`.
- `logistic(x) := 1 / (1 + e^(-x))` — recognized from forms like
  `0.9 / (1.0 + exp(-2*x - 1.5*y))` (→ `0.9*logistic(2*x + 1.5*y)`). Disable with
  `cvt:pre:logistic=0`.

**Piecewise-linear approximation.** For linear-quadratic MP-based solvers, most of these
nonlinear functions (except products with binary variables) are handled by
piecewise-linear approximation, controlled by `cvt:plapprox:reltol` and
`cvt:plapprox:domain`. Tight variable bounds reduce approximation size and improve
numerical behavior — same recommendation as for the `<<...>>` operator above.

**Handling in Gurobi.** Gurobi natively handles the univariate functions `exp`, `log`,
`^`, `sin`, `cos`, `tan`, `tanh` as part of its MINLP capability (global nonlinear solving
via spatial branching); Gurobi 12+ defaults to this proper nonlinear handling rather than
piecewise-linear approximation (the older behavior, deprecated in Gurobi 13). To force
piecewise-linear approximation instead:

```ampl
ampl: option gurobi_options 'global=-1 acc:_expr=0'; solve;
```

or, per constraint/objective, via the `.funcnonlinear`/`.global` suffix (`pre:funcnonlinear`:
-1 forces PL approximation, 0 automatic (default), 1 forces nonlinear treatment). When PL
approximation is in effect, tune it via `gurobi_options` `pre:funcpieces`,
`pre:funcpiecelength`, `pre:funcpieceerror`, `pre:funcpieceratio` — or per-objective/
constraint with suffixes of the same names:

```ampl
minimize Chichinadze:
   x[1]^2 - 12*x[1] + 11 + 10*cos(pi*x[1]/2) +
      8*sin(pi*5*x[1]) - exp(-(x[2]-.5)^2/2)/sqrt(5);
```

`suffix funcpieces IN; let Chichinadze.funcpieces := 12;` sets 12 approximation pieces for
this objective's `sin`/`cos`/`exp` terms specifically.

**This section updates `../modeling-patterns/nonlinear-programs.md`** for anything
solver-interface-related — the modeling-patterns file's NLP pitfalls (local optima,
starting-point sensitivity) still apply, but MP's native/piecewise-linear-approximation
handling of these functions, and the `signpow`/`logistic` recognition above, are MP-layer
behavior not covered there.

## See also

- `overview-and-when-to-use.md` — what MP is and which solvers use it
- Piecewise-linear notation, detection rules, and non-`<<...>>`-expressible cases:
  `../modeling-patterns/piecewise-linear-programs.md`
- Complementarity: full derivation, satisfaction rules, suffixes:
  `../modeling-patterns/complementarity-problems.md`
- NLP-specific pitfalls (multiple local optima, starting points):
  `../modeling-patterns/nonlinear-programs.md`
- Bounding subexpressions and variables for solver performance and numerical stability:
  `efficiency-and-numerical-accuracy.md`
- `acc:` acceptance options controlling native vs. reformulated handling per constraint
  type, and exporting the solver's model to see what reformulation actually happened:
  `tools-troubleshooting-and-reformulation.md`
