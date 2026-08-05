# AMPL MP Expressions Reference

The MP solver interface library allows AMPL expressions to be used in more general ways and with
a greater variety of solvers. MP-based solvers handle these expressions natively or via automatic
reformulation -- there is **no need to manually linearize** with big-M or other tricks.

**All current AMPL solvers are MP-based:** Gurobi, CPLEX, COPT, Xpress, Mosek, BARON, HiGHS,
cuOpt, CBC, SCIP, GCG, plus the MP2NL meta-solver.

## Conditional Operators

### if-then-else (expr-valued)

`if constr then expr1 [else expr2]` -- when constr is true, takes the value of expr1. When
false, takes expr2 (or 0 if else is omitted). The constr may contain variables.

```ampl
minimize TotalCost:
   sum {j in JOBS, k in MACHINES}
      if MachineForJob[j] = k then cost[j,k];

subject to Balance {p in PROD, t in TIME}:
   Make[p,t] + (if t = 0 then inv0[p] else Inv[p,t-1])
      = Sell[p,t] + Inv[p,t];
```

### Implication operators (constr-valued)

- `constr1 ==> constr2 [else constr3]` -- if constr1 true, then constr2 must hold; if constr1
  false, constr3 must hold (if present).
- `constr2 <== constr1` -- same as `constr1 ==> constr2`.
- `constr1 <==> constr2` -- both true or both false.

When constr1 is of the form `binary_var = 0` or `binary_var = 1`, these become **indicator
constraints** handled natively by many solvers (Gurobi, CPLEX, COPT, Xpress).

```ampl
subject to Multi_Min_Ship {i in ORIG, j in DEST}:
   sum {p in PROD} Trans[i,j,p] > 0 ==>
      minload <= sum {p in PROD} Trans[i,j,p] <= limit[i,j];

subject to Least_Use {j in SCHEDS}:
   Use[j] = 1 ==> Work[j] >= least_assign else Work[j] = 0;
```

## Logical Operators

- `constr1 or constr2` -- satisfied when either is true.
- `constr1 and constr2` -- satisfied when both are true.
- `not constr` -- satisfied when constr is false.
- `exists {indexing} constr` -- satisfied when at least one operand is true (iterated `or`).
- `forall {indexing} constr` -- satisfied when all operands are true (iterated `and`).

```ampl
# Disjunctive scheduling -- no big-M needed
subj to No_Conflict {i1 in JOBS, i2 in JOBS: ord(i1) < ord(i2)}:
   Start[i2] >= Start[i1] + t_offset[i1,i2] or
   Start[i1] >= Start[i2] + t_offset[i2,i1];

subject to Least_Use {j in SCHEDS}:
   Work[j] = 0 or least_assign <= Work[j] <= max {i in SHIFT_LIST[j]} required[i];

subj to EntRem {t in 1..numTanks}:
   Entry[t] + minTime[t] <= Removal[t] and
   Entry[t] + maxTime[t] >= Removal[t];

# Fixed cost using exists
minimize Total_Cost:
   sum {p in PRODUCTS, (i,j) in ARCS} var_cost[p,i,j] * Flow[p,i,j] +
   sum {(i,j) in ARCS} if exists {p in PRODUCTS} Flow[p,i,j] > 0 then fix_cost[i,j];

# Iterated logical operators
subject to Multi {i in ORIG, j in DEST}:
   forall {p in PROD} Trans[i,j,p] = 0 or
   minload <= sum {p in PROD} Trans[i,j,p] <= limit[i,j];
```

## Piecewise-Linear Functions

### abs, min, max

These are handled natively by solvers that support them (Gurobi ABS/MIN/MAX general
constraints), or linearized automatically for others.

```ampl
maximize Total_Profit:
   sum {p in PROD, t in 1..T} revenue[p,t]*Sell[p,t] -
   sum {t in 1..T} time_penalty[t] * abs(Use[t] - avail_min[t]);

minimize Max_Cost:
   max {i in PEOPLE} sum {j in PROJECTS} cost[i,j] * Assign[i,j];

maximize WeightSum:
   sum {t in TRAJ} max {n in NODE} weight[t,n] * Use[n];
```

### Piecewise-linear operator

`<< breakpoint-list; slope-list >> var` -- defined by n breakpoints and n+1 slopes.

**Important:** Set `option pl_linearize 0;` for MP-based solvers so they handle PL expressions
natively (Gurobi, COPT) or via the MP interface (HiGHS, others).

```ampl
maximize Total_Profit:
   sum {p in PROD, t in 1..T} (revenue[p,t]*Sell[p,t] -
      prodcost[p]*Make[p,t] - <<0; -backcost[p],invcost[p]>> Inv[p,t]) -
   sum {t in 1..T} <<avail_min[t]; 0,time_penalty[t]>> Use[t];

minimize Total_Cost:
   sum {i in ORIG, j in DEST}
      <<{p in 1..npiece[i,j]-1} limit[i,j,p];
        {p in 1..npiece[i,j]} rate[i,j,p]>> Trans[i,j];
```

## Counting Operators

### count
`count {indexing} constr` -- the number of members where constr is satisfied.

```ampl
subject to Min_Serve {i in ORIG}:
    count {j in DEST} (Ship[i,j] >= minload) >= minserve;
```

### atleast, atmost, exactly
- `atleast k {indexing} constr` -- constr satisfied for at least k members.
- `atmost k {indexing} constr` -- constr satisfied for at most k members.
- `exactly k {indexing} constr` -- constr satisfied for exactly k members.

```ampl
subject to Min_Serve {i in ORIG}:
    atleast minserve {j in DEST} (Ship[i,j] >= minload);

subj to CapacityOfMachine {k in MACHINES}:
    atmost cap[k] {j in JOBS} (MachineForJob[j] = k);
```

### numberof
`numberof expr in (expr-list)` -- count of items in the list equal to expr.

```ampl
subj to CapacityOfMachine {k in MACHINES}:
    numberof k in ({j in JOBS} MachineForJob[j]) <= cap[k];
```

### alldiff
`alldiff {indexing} expr` -- all values must be different.

```ampl
subject to OnePersonPerPosition:
   alldiff {i in 1..nPeople} Pos[i];

subject to Regions {I in 1..9 by 3, J in 1..9 by 3}:
   alldiff {i in I..I+2, j in J..J+2} X[i,j];
```

## Comparison Operators

`!=` works directly in constraints -- MP handles the reformulation.

```ampl
subject to Different_Colors {(c1,c2) in Neighbors}:
   Color[c1] != Color[c2];
```

## Complementarity Operator

`constr1 complements constr2` -- both satisfied, and at least one holds with equality.

```ampl
subject to Pri_Compl {i in PROD}:
   Price[i] >= 0 complements
      sum {j in ACT} io[i,j] * Level[j] >= demzero[i] - demrate[i] * Price[i];
```

## Set Membership Operator

`var X in set-expr;` -- variable must take a value from the specified set.

```ampl
var Buy {f in FOODS} in {0, 10, 30, 45, 55};
var Ship {(i,j) in ARCS} in {0} union interval[min_ship, capacity[i,j]];
```

## Quadratic and Power Operators

Products of linear expressions and `^` for powers are supported. For convex QP, solvers handle
these natively. For non-convex cases, MP can linearize via piecewise-linear approximation.

```ampl
minimize Portfolio_Variance:
   sum {i in ASSETS, j in ASSETS} covariance[i,j] * w[i] * w[j];
```

## Conic Optimization

SOCP constraints are recognized from algebraic form by Mosek, Gurobi, COPT, SCIP, CPLEX, Xpress.

```ampl
# Standard SOC
s.t. cone: x[0] >= sqrt(x[1]^2 + x[2]^2 + x[3]^2);

# Rotated SOC (x[0], x[1] >= 0)
s.t. rotated: 2*x[0]*x[1] >= x[2]^2 + x[3]^2;
```

Mosek also handles exponential cones natively:
```ampl
s.t. T1: 1 + b*w >= exp(q1);
```

## General Nonlinear Functions

Supported: `exp`, `log`, `log10`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `sinh`, `cosh`,
`tanh`, `asinh`, `acosh`, `atanh`, and `^` with variable exponents.

For linear/MIP solvers, these are approximated via piecewise-linear functions automatically.
For Gurobi 12+, native nonlinear handling via spatial branching.

```ampl
minimize Chichinadze:
   x[1]^2 - 12*x[1] + 11 + 10*cos(pi*x[1]/2) +
      8*sin(pi*5*x[1]) - exp(-(x[2]-.5)^2/2)/sqrt(5);
```

## Key Reformulation Options

- `option pl_linearize 0;` -- let MP handle piecewise-linear (recommended for MP solvers)
- `cvt:mip:eps` -- tolerance for strict inequalities (e.g., `x > 5` becomes `x >= 5 + eps`)
- `cvt:quadobj`, `cvt:quadcon` -- control quadratic reformulation
- `cvt:socp`, `cvt:socp2qp` -- control SOCP reformulation
- `cvt:compl` -- control complementarity reformulation
