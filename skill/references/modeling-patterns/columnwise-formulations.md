# Columnwise Formulations: `to_come`, `coeff`, `obj`, `cover`

Derived from: ampl-book.md (ch. 16)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 16.

## When to reach for this

AMPL's default orientation is row-wise: declare variables first, then objective and
constraints that reference them. That's natural when a single *constraint's* coefficients
across all variables have a simple pattern or interpretation. Some models are the mirror
image: a single *variable's* coefficients down all the constraints it touches have the
simple, describable pattern — e.g. one activity/schedule/route contributing to a handful
of named constraints. For those, declare the objective and constraints first (as empty
templates), then attach each variable's coefficients to it directly in its own `var`
declaration. This is the same idea already introduced for networks in
`network-models.md` (`node`/`arc` are literally a specialized, always-available case of
columnwise declaration) — this file generalizes the mechanism to any linear program, not
just network-shaped ones.

Rule of thumb from the source material: columnwise formulations shine when the model's
natural unit of thought is "one variable and everything it touches" (one production
activity, one weekly work schedule, one cutting pattern) — especially when new instances
of that unit get added or edited as data changes, without a code change. Row-wise stays
preferable when the model has many distinct, specialized constraint groups that don't
reduce to a single simple per-variable pattern.

## Mechanism: `to_come` templates + `coeff`/`obj` phrases

Declare the objective and constraints with `to_come` standing in for "the linear terms
that will show up here, contributed by variables declared later":

```ampl
maximize Net_Profit;                                # to_come is implicit/omittable when it's the WHOLE objective
subject to Balance {i in MAT}: to_come = 0;          # to_come + const-expr, here just "= 0"
```

Then each `var` declaration supplies its own coefficients via `obj`/`coeff` phrases:

```ampl
var Run {j in ACT} >= act_min[j], <= act_max[j],
   obj Net_Profit revenue[j],                        # term revenue[j]*Run[j] added to Net_Profit
   coeff {i in MAT} Balance[i] io[i,j];               # term io[i,j]*Run[j] added to each Balance[i]
```

Because `obj`/`coeff` phrases *reference* the objective/constraint names, the `var`
declarations using them must come **after** the `maximize`/`minimize`/`subject to`
declarations they refer to — the opposite of AMPL's usual row-wise ordering.

- `to_come` alone, `to_come + arith-expr`, or `arith-expr + to_come` are the three legal
  template forms; a template constraint's non-`to_come` side must be a `const-expr` (no
  variables) in the ordinary case — the same `to_come` mechanism used for network
  side-constraints in `network-models.md`. If the template's other side *does* contain
  variables, they must come from `var` declarations already read, producing a hybrid
  row/columnwise model — legal but reduces the clarity columnwise formulations are meant
  to provide.
- If the objective is *entirely* built from `obj` phrases in later `var` declarations,
  the `: expression` part of the objective declaration can be omitted — bare
  `maximize Net_Profit;` — since `: to_come` is the implicit default.
- `coeff [indexing] ConstraintName arith-expr` — the indexing expression (if present)
  generates one coefficient per member, exactly like a `sum`'s indexing; the special
  `{if lexpr}` form (also used for `node`/`arc` and `{if lexpr}` constraint indexing,
  `05-variables-objectives-constraints.md`) conditionally includes a coefficient. One
  `coeff` phrase per distinct *indexed collection* of constraints the variable
  participates in — a variable touching two unrelated constraint families needs two
  `coeff` phrases.
- Any constraint not named in a `coeff` phrase gets an implicit coefficient of 0 for that
  variable — you only ever write down the nonzeros, which is the entire point when most
  variable/constraint pairs are structurally zero.
- `cover [indexing] ConstraintName` is shorthand for `coeff [indexing] ConstraintName 1` —
  handy for the extremely common case (scheduling, set-covering) where a variable simply
  "covers" a constraint with weight 1, paralleling how `from`/`to` implicitly mean weight
  ±1 for network arcs.

## Worked contrast: input-output production model

Row-wise (natural until the model needs refinement — see below):

```ampl
param io {MAT,ACT};                          # io[i,j] > 0: material i produced by activity j; < 0: consumed
var Run {j in ACT} >= act_min[j], <= act_max[j];
maximize Net_Profit: sum {j in ACT} revenue[j] * Run[j];
subject to Balance {i in MAT}: sum {j in ACT} io[i,j] * Run[j] = 0;
```

Columnwise — same LP, but all of activity `j`'s data (bounds, revenue, input-output
column) is now assembled in one place, which is exactly how a modeler thinks about
"adding one more activity":

```ampl
maximize Net_Profit;
subject to Balance {i in MAT}: to_come = 0;
var Run {j in ACT} >= act_min[j], <= act_max[j],
   obj Net_Profit revenue[j],
   coeff {i in MAT} Balance[i] io[i,j];
```

**Where columnwise pays off most**: when the model is later refined to split out sale
activities as their own variable (`Sell{MATF}`), the row-wise version needs an
`if-then-else` inside the `Balance` constraint (`= if i in MATF then Sell[i] else 0;`) —
awkward, and it obscures the underlying LP structure. The columnwise version needs no
constraint change at all — just a second `var` declaration with its own `obj`/`coeff`
phrases:

```ampl
var Sell {i in MATF} >= sell_min[i], <= sell_max[i],
   obj Net_Profit revenue[i],
   coeff Balance[i] -1;                       # a "sale" is a pure input of material i, hence -1
```

## Worked contrast: shift-scheduling / set-covering

A schedule `j` is a subset of shifts (`SHIFT_LIST[j]`); a person on that schedule covers
every shift in the subset. Row-wise requires an awkward "is `i` in this schedule"
membership sum inside the constraint:

```ampl
subject to Shift_Needs {i in SHIFTS}:
   sum {j in SCHEDS: i in SHIFT_LIST[j]} Work[j] >= required[i];
```

Columnwise states the same relationship the way the data is naturally organized (by
schedule, listing which shifts it covers):

```ampl
minimize Total_Cost;
subject to Shift_Needs {i in SHIFTS}: to_come >= required[i];
var Work {j in SCHEDS} >= 0,
   obj Total_Cost rate[j],
   cover {i in SHIFT_LIST[j]} Shift_Needs[i];       # coeff ... 1, via the cover shorthand
```

This is the classic **set covering** pattern: minimize cost of selected subsets (crew
schedules, cutting patterns, work shifts) such that every member of a base set (flights,
stock orders, shift-slots) is covered by at least one selected subset. Airline crew
scheduling is the best-known large-scale instance — variables represent candidate crew
pairings/schedules rather than individuals, "shifts" become flights. Relaxed-LP optimal
solutions to covering problems are frequently fractional (as in the worked example: 13 of
126 candidate schedules used, several at fractional levels) — getting a genuine
whole-number staffing plan requires integer programming
(`integer-programming.md`), though naive "round every fraction up" often gives a
usably-close feasible answer for a first pass.

## Columnwise side constraints on a network model

`coeff`/`obj` phrases can be added to `arc` declarations alongside the usual `from`/`to`,
letting you describe a *non-network* side constraint (e.g. a joint capacity shared across
several network copies, as in the multicommodity-flow example in `network-models.md`) in
the same columnwise style, without falling back to a purely algebraic `subject to`:

```ampl
subject to Multi {(i,j) in LINKS}: to_come <= cap_joint[i,j];
arc Ship {(i,j) in LINKS, p in PRODS} >= 0, <= capacity[i,j,p],
   from Balance[i,p], to Balance[j,p],
   coeff Multi[i,j] 1.0,
   obj Total_Cost cost[i,j,p];
```

## See also

- `node`/`arc` as the network-specific special case of this same columnwise mechanism:
  `network-models.md`
- `{if lexpr}` conditional-indexing form used identically here, in constraint
  declarations, and in `node`/`arc`: `05-variables-objectives-constraints.md`
- Integer/binary formulations of set-covering and cutting-stock problems:
  `integer-programming.md`
- Multiperiod replication (inventory-tying-periods pattern), which the exercises in this
  chapter suggest combining with columnwise declarations: `classic-lp-patterns.md`
