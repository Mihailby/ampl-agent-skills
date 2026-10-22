# Complementarity Problems: `complements`

Derived from: ampl-book.md (ch. 19)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 19.

## Table of contents

- What complementarity expresses, and why it's not just "another constraint type"
- From LP duals to complementarity: the production-economics equilibrium example
- Bounded-variable and price-dependent-demand extensions
- The `complements` operator: exact forms and satisfaction rules
- Working with complementarity constraints: `.Lsuf`/`.Rsuf` suffixes, presolve, generic
  synonyms, `solexpand`'s canonical form

## What complementarity expresses

A complementarity condition links two inequalities: both must hold, and **at least one
must hold with equality**. This shows up naturally whenever an equilibrium is being
modeled rather than an optimum — "either this resource is scarce (its price is positive)
or it's not fully used (its slack is positive), never both at once" is exactly a
complementarity statement, not an optimization objective. In fact, the optimality
conditions of any LP or smooth NLP *are themselves* a complementarity problem — so
complementarity is a strict generalization of optimization, useful specifically for models
that describe an equilibrium state (economic, physical, game-theoretic) with no natural
objective function to optimize, or where the "equivalent" objective (if one even exists)
is inconvenient to derive and solve directly.

The AMPL `complements` operator lets these conditions be written directly, in close
correspondence to how an economist or physicist already writes them on paper, rather than
forcing a translation into either an optimization objective or a manually-encoded
big-M/indicator formulation.

## From optimization to equilibrium: a worked derivation

Start from an ordinary LP — minimize production cost subject to meeting demand:

```ampl
var Level {j in ACT} >= 0;
minimize Total_Cost: sum {j in ACT} cost[j] * Level[j];
subject to Demand {i in PROD}: sum {j in ACT} io[i,j] * Level[j] >= demand[i];
```

The LP's dual values (`Demand[i].dual`) are shadow prices. Reformulating the *same*
economics as an equilibrium (no objective — the model just asserts the equilibrium
conditions must hold) introduces explicit `Price` variables and states two
complementarity relationships directly:

```ampl
var Price {i in PROD};
var Level {j in ACT};

subject to Pri_Compl {i in PROD}:
   Price[i] >= 0 complements
      sum {j in ACT} io[i,j] * Level[j] >= demand[i];

subject to Lev_Compl {j in ACT}:
   Level[j] >= 0 complements
      sum {i in PROD} Price[i] * io[i,j] <= cost[j];
```

Read `Pri_Compl` as: "price is nonnegative, output meets demand, and at least one holds
with equality" — i.e. if output *exceeds* demand, the price must be zero (no scarcity, no
value); if the price is positive, output exactly meets demand. `Lev_Compl` reads
similarly for activity levels vs. cost/value. Solving this square system (with a
complementarity-capable solver such as PATH, not a standard LP solver — see below)
reproduces exactly the LP's primal (`Level`) and dual (`Demand.dual`) values — confirming
the equilibrium view and the optimization view describe the same economics, when the
underlying problem is linear.

**Why go to the trouble** when the LP already gives the same answer? Two of this
chapter's own extensions show why: bounded activity levels, and *price-dependent* demand.
Both remain natural, solvable complementarity problems; only the first has a
straightforward LP-equivalent reformulation, and the second has none at all.

### Bounded variables: complementarity with a double inequality

Adding bounds `level_min[j] <= Level[j] <= level_max[j]` requires a **mixed**
complementarity form — a double inequality complementary to a single expression:

```ampl
subject to Lev_Compl {j in ACT}:
   level_min[j] <= Level[j] <= level_max[j] complements
      cost[j] - sum {i in PROD} Price[i] * io[i,j];
```

Reads as: at the lower bound, the expression (cost minus value) may be positive (not
worth increasing); at the upper bound, it may be negative (not worth decreasing);
strictly between the bounds, it must be exactly zero. This is the standard
KKT-complementary-slackness condition for a bounded variable, spelled out directly.

### Price-dependent demand: genuinely NOT an LP anymore

Replacing the fixed `demand[i]` with a price-responsive function —
`demzero[i] - demrate[i] * Price[i]` — keeps the problem a well-posed, solvable
complementarity system, but because `Price[i]` now appears on **both** sides of a
`complements` relationship, there is no equivalent linear program (and no easy-to-derive
equivalent nonlinear program either). This is the clearest illustration of why
`complements` earns its own modeling vocabulary rather than being treated as sugar for an
optimization reformulation — some genuinely useful equilibrium models simply have no
natural objective-function form.

### Other applications (recognize the pattern)

- **Physical contact/force equilibria**: one side of `complements` holds with equality
  where two objects touch, the other side where they don't.
- **Nash equilibria in bimatrix games**: for each pure strategy, either its probability is
  zero, or its expected payoff equals the player's maximum achievable payoff — a direct
  complementarity statement over strategy probabilities.
- **MPECs** (mathematical programs with equilibrium constraints — an objective mixed with
  equality/inequality/complementarity constraints in arbitrary combination): supported by
  AMPL's syntax, but solution techniques for the general case are described as
  experimental in the source material — treat as an advanced/specialist case.

## `complements`: exact syntactic forms and satisfaction rules

**Single inequality vs. single inequality:**
```ampl
single-inequality complements single-inequality;
```
Satisfied iff both inequalities hold, and at least one holds with equality. Each side may
be linear or nonlinear, and must contain exactly one `>=` or `<=`.

**Double inequality vs. an expression** (either order):
```ampl
double-inequality complements expression;
expression complements double-inequality;
```
`double-inequality` has two `>=` (or two `<=`); `expression` is any numeric expression
(variables may appear nonlinearly on either side). Satisfaction rule:
- lower bound active (or upper-side `>=` active) → expression `>= 0`
- upper bound active (or lower-side `>=` active) → expression `<= 0`
- neither bound active → expression `= 0`

The special case `0 <= body <= Infinity complements ...` reduces exactly to the
single-inequality-pair rule above — bounded and unbounded complementarity are the same
mechanism, just with `Infinity` standing in for "no bound" (`12-syntax-quick-reference.md`,
section A.7.2 on `Infinity`).

**Degenerate equality case** (either order):
```ampl
equality complements expression;
```
Places no restriction on `expression` at all — equivalent to an ordinary equality
constraint. Included mainly so that "make this square" conversions (below) have a
uniform rule to fall back on.

### "Square" systems and solvers

Complementarity-specific solvers (PATH is the example used throughout the source
chapter) generally require a **square** system: number of variables equals number of
complementarity constraints plus equality constraints, with any remaining inequality
constraints (and no objective) allowed as long as squareness is preserved — AMPL
automatically pairs each unmatched inequality with a new otherwise-unused variable to
keep the system square when converting for such a solver. Set the solver explicitly:
`option solver path;` before `solve`.

## Working with complementarity constraints

### `.Lsuf` / `.Rsuf` suffixes and `.slack`

Every ordinary suffix (`.lb`, `.body`, `.ub`, `.slack`, ...) extends to complementarity
constraints via a **left/right pair**: `cname.Lbody`, `cname.Rbody`, `cname.Lslack`,
`cname.Rslack`, etc., referring to the left and right operand of `complements`
respectively (an unadorned `expression` side is treated as a "constraint" with infinite
bounds and hence infinite slack). An unsuffixed `cname` in an expression is interpreted as
`cname.slack` — nonnegative when both sides hold, zero exactly when the complementarity
condition is satisfied exactly, letting you write generic satisfaction checks without
hand-deriving the sign convention:

```ampl
display Lev_Compl.Llb, Lev_Compl.Lbody, Lev_Compl.Rbody, Lev_Compl.Rslack;
```

Full formal `.slack` sign rules for both the single- and double-inequality forms are in
`12-syntax-quick-reference.md` (Table A-7 area / section A.11).

### Presolve simplifications specific to complementarity

Presolve can deduce, e.g., that one side of a complementarity pair is *always* strictly
positive across the feasible region, and simplify the constraint to a plain equality on
the other side — or, symmetrically, that one side is always negative, letting a bounded
variable be fixed at a specific bound and removed entirely. This is a genuine size
reduction, not just a display convenience — worth checking `option show_stats 1;` output
(`Presolve resolves N of M complementarity conditions.`) on a large complementarity model,
since it directly affects solver difficulty.

Diagnostic parameters, before/after presolve:

| Parameter | Meaning |
|---|---|
| `_ncons` / `_nccons` | ordinary / complementarity constraint counts before presolve |
| `_sncons` / `_snccons` | ordinary / complementarity constraint counts after presolve |

Before presolve, AMPL counts each complementarity constraint as *both* one complementarity
condition *and* two ordinary constraints (its two `complements` operands) — so
`_ncons = 2 * _nccons + (non-complementarity constraint count)`.

### Generic synonyms: `_ccon` family, and `.L`/`.R` naming

Parallel to the ordinary `_con`/`_conname`/`_var` families
(`09-display-and-output-commands.md`), complementarity constraints get their own
pre-presolve family — `_nccons`, `_cconname`, `_ccon` — and each complementarity
constraint `cname` also contributes two entries to the *ordinary* `_conname`/`_con`
arrays, suffixed `.L` and `.R` for its left/right operand:

```ampl
expand Pri_Compl['AA1'];              # the complementarity constraint as written
expand _con[1], _con[2];              # its .L and .R halves as separate ordinary constraints
```

**Solver's view (`solexpand`)**: AMPL canonicalizes every complementarity constraint to
one of three forms before sending it to the solver —
`expr complements lbound <= var <= ubound`, `expr <= 0 complements var <= ubound`, or
`expr >= 0 complements lbound <= var` — introducing an auxiliary variable and defining
equality when the constraint's "variable side" is actually a more complex expression.
`_scvar[i]` gives the index (into `_svar`/`_svarname`) of the complementing variable for
solver-view constraint `i`, or `0` for an ordinary (non-complementarity) constraint —
useful for writing generic post-solve reports over an arbitrary complementarity model
without hard-coding constraint names.

## See also

- Dual values / shadow prices as the LP special case complementarity generalizes:
  `09-display-and-output-commands.md`
- Presolve mechanics in general (the complementarity-specific rules here extend the same
  machinery): `11-solver-interaction-and-suffixes.md`
- `_con`/`_var` generic synonym families that the `_ccon` family parallels:
  `09-display-and-output-commands.md`
- `Infinity` as the "no bound" sentinel used in the single-inequality reduction case:
  `12-syntax-quick-reference.md` (section A.7.2)
