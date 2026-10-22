# Variables, Objectives, and Constraints (Linear Programs)

Derived from: ampl-book.md (ch. 8)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 8.

## Variable declarations

Syntax mirrors `param`, but with `var`:

```ampl
var Make {p in PROD} >= 0, <= market[p];
```

- `>=`/`<=` phrases are bounds, may be any expression in previously-defined sets/params/dummy
  indices — this is how nonnegativity (`>= 0`) is normally expressed.
- **`<`, `>`, `<>` are not allowed** on variable declarations — strict inequalities are
  meaningless for continuous variables (there's always a value arbitrarily close to the bound).
- Bounds in the `var` declaration vs. as a separate `subject to` constraint solve to the same
  answer, but many solvers handle declared bounds specially/faster, and the model reads more
  clearly with bounds inline. Prefer putting bounds directly in the `var` declaration.
- **`= expr`** creates a *defined variable*: the expression (which, unlike a parameter's `=`,
  may itself reference other variables) is substituted everywhere the defined variable is used;
  it does **not** add a constraint. Use this to name and expose intermediate quantities for
  readability and for `display`, without changing the problem:
  ```ampl
  var Total_Revenue = sum {p in PROD, t in 1..T} sum {a in AREA[p]} revenue[p,a,t]*Sell[p,a,t];
  var Total_Prod_Cost = sum {p in PROD, t in 1..T} prodcost[p]*Make[p,t];
  maximize Total_Profit: Total_Revenue - Total_Prod_Cost;
  ```
  If the right-hand side contains no variables at all, you actually want `param`, not `var`. If
  you just want to temporarily pin a variable's value while exploring/debugging, use the `fix`
  command (`08-commands-modifying-models-and-data.md`) instead of rewriting the declaration.
- `:=` or `default` gives an **initial value** (a starting point for the solver) — rarely useful
  for LP solvers, much more relevant for nonlinear solvers (see `nonlinear-programs.md`).
- `integer` / `binary` restrict a variable to whole numbers / {0,1} — this makes the model an
  integer program; see `integer-programming.md` for solver implications.

## What counts as "linear" to AMPL

A linear expression is built entirely from:

```
constant-expr
var-ref
(constant-expr) * (linear-expr)
(linear-expr) * (constant-expr)
(linear-expr) / (constant-expr)
if logical-expr then linear-expr else linear-expr    # ok IF the condition itself has no variables
```

where `constant-expr` contains no variable references. Consequences worth internalizing:

- Division **by a variable** is not linear; division of a linear expression by a constant is
  fine (`Make[p,t] / rate[p]` is linear because `rate[p]` is a parameter).
- Variables may **not** appear as arguments to `min`, `max`, `abs`, `sqrt`, `log`, trig
  functions, `^`, or iterated `forall`/`exists` — using any of these on a variable makes the
  expression nonlinear to AMPL (even if mathematically it could be linearized). See
  `nonlinear-programs.md`, or reformulate:
  ```ampl
  # NOT linear -- division by a sum of variables
  subject to Limit {p in PROD}: Make[p] / sum {q in PROD} Make[q] <= max_frac;
  # linear reformulation -- multiply the bound across instead
  subject to Limit {p in PROD}: Make[p] <= max_frac * sum {q in PROD} Make[q];
  ```
  This "clear the denominator" trick is the standard fix whenever a ratio-of-variables
  constraint isn't recognized as linear.
- Multiplication/division distributes over `sum`, so unit-conversion factors can be placed
  either outside the whole sum or inside each term — pick whichever reads best; the results are
  numerically identical modulo floating point.
- AMPL canonicalizes automatically (combines like terms, moves things across `<=`/`=`) — you
  don't need variables on one particular side. Use `expand ConstraintName;` to see the
  canonical form AMPL actually generated (`01-language-basics.md`).

## Objectives

```ampl
minimize Total_Cost: sum {j in FOOD} cost[j] * Buy[j];
maximize Total_Profit: sum {p in PROD} profit[p] * Make[p];
```

- A model *may* declare more than one objective (including an **indexed collection** of
  objectives, e.g. `minimize Pref_of {i in ORIG}: sum {j in DEST} cost[i,j]*Trans[i,j];`), but
  only one is active for any given `solve`. AMPL uses the first one declared unless told
  otherwise:
  ```
  ampl: objective Total_NA;          # switch active objective before the next solve
  ampl: objective Pref_of["Coullard"];  # select one member of an indexed objective collection
  ampl: solve;
  ```
- The objective's name, outside of columnwise-formulation contexts
  (`../modeling-patterns/columnwise-formulations.md`), is otherwise just a handle: it refers to
  the objective's *value* in commands (`display Total_Cost;`).
- **Multi-objective exploration pattern**: pin the first objective's value with an equality
  constraint, then optimize a second objective within that constraint — lets you explore among
  alternative optima without sacrificing the primary goal:
  ```ampl
  subject to Stay_Optimal: sum {i in ORIG, j in DEST} cost[i,j]*Trans[i,j] = 28;
  minimize Pref_of {i in ORIG}: sum {j in DEST} cost[i,j]*Trans[i,j];
  ```
  This is the standard technique for "find the best solution for person/unit X among all
  solutions that are still optimal overall" — useful whenever a problem (like assignment) has
  multiple optimal solutions and you want to break ties by a secondary criterion.

## Constraints

```ampl
subject to Time: sum {p in PROD} (1/rate[p]) * Make[p] <= avail;
```

- `subject to` is optional — any declaration not starting with a recognized keyword is
  interpreted as a constraint — but always write it for readability.
- Indexed constraint families follow the same rules as indexed params/vars: `Time[1]`,
  `Balance[p, t+1]` refer to individual members.
- **Conditional constraint generation**: append a filter to the indexing expression to skip
  generating a constraint for certain members entirely (different from bounding it — the
  constraint simply doesn't exist for filtered-out members):
  ```ampl
  subject to Time {t in 1..T: avail[t] > 0}: sum {p in PROD} (1/rate[p])*Make[p,t] <= avail[t];
  subject to Time {if avail > 0}: sum {p in PROD} (1/rate[p]) * Make[p] <= avail;  # unindexed case
  ```
- **Double-inequality (range) constraints** bound one linear expression from both sides in a
  single declaration — the two outer bounds must themselves be variable-free:
  ```ampl
  const-expr <= linear-expr <= const-expr
  const-expr >= linear-expr >= const-expr
  ```
  If you need a variable-dependent bound on either side, that's two separate constraints, not
  one double inequality.
- AMPL applies **presolve** when handing the model to a solver (fixing/eliminating variables,
  folding single-variable constraints into bounds, dropping implied-redundant constraints) —
  usually invisible, but see `11-solver-interaction-and-suffixes.md` for how to inspect or
  disable it when a solve behaves unexpectedly.

## See also

- Debugging with `expand`, and the `.dual`/`.rc`/`.slack` suffixes: `01-language-basics.md`,
  `11-solver-interaction-and-suffixes.md`
- Modifying variables/objective selection/data interactively (`let`, `fix`, `objective`,
  `drop`/`restore`): `08-commands-modifying-models-and-data.md`
- Nonlinear reformulations when a natural formulation isn't linear: `nonlinear-programs.md`
- Whole-number/binary requirements: `integer-programming.md`
