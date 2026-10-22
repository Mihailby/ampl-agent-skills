# Classic LP Formulation Patterns

Derived from: ampl-book.md (ch. 1-4)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapters 1-4.
These are the "load-bearing" LP patterns that most real applications are built from or
combined out of. For AMPL syntax fundamentals (sets/params/vars/objective/constraints),
see `../core-language/01-language-basics.md` first — this file assumes that vocabulary and
focuses on *which formulation to reach for*.

## Pattern 1: Production / resource-allocation (maximize profit subject to capacity)

Use when: choosing production levels of several products that compete for shared, limited
resources (time, capacity, budget), each with a per-unit profit.

```ampl
set PROD;                       # products
param rate {PROD} > 0;          # units produced per hour
param avail >= 0;               # hours available
param profit {PROD};            # profit per unit
param market {PROD} >= 0;       # upper limit on units sold
var Make {p in PROD} >= 0, <= market[p];
maximize Total_Profit: sum {p in PROD} profit[p] * Make[p];
subject to Time: sum {p in PROD} (1/rate[p]) * Make[p] <= avail;
```

Common extensions:

- **Lower bounds / sales commitments** — change the variable's lower bound from `0` to a
  `commit[p]` parameter: `var Make {p in PROD} >= commit[p], <= market[p];`
- **Multiple resource/production stages** — add a `STAGE` set and index `rate`/`avail` over it,
  turning the single `Time` constraint into a family `subject to Time {s in STAGE}: ...`. This
  is the general recipe for "the same decision competes for several different limited
  resources" (labor hours *and* machine hours *and* budget, for example).
- Read `Make.rc` (reduced cost) and the constraint's dual value (e.g. `display Time;`) to see
  which bound or resource is actually binding — this is usually more informative to a
  stakeholder than the raw optimal solution.

## Pattern 2: Diet / blending / minimum-cost input selection

Use when: choosing quantities of several "inputs" (foods, raw materials, feedstocks, work
schedules) to satisfy lower/upper bounds on several "output" qualities (nutrients, blend
specs, staffing coverage) at minimum cost.

```ampl
set NUTR;
set FOOD;
param cost {FOOD} > 0;
param f_min {FOOD} >= 0;
param f_max {j in FOOD} >= f_min[j];
param n_min {NUTR} >= 0;
param n_max {i in NUTR} >= n_min[i];
param amt {NUTR,FOOD} >= 0;      # amount of nutrient i per unit of food j
var Buy {j in FOOD} >= f_min[j], <= f_max[j];
minimize Total_Cost: sum {j in FOOD} cost[j] * Buy[j];
subject to Diet {i in NUTR}:
   n_min[i] <= sum {j in FOOD} amt[i,j] * Buy[j] <= n_max[i];
```

The double-inequality constraint (`lb <= expr <= ub`) is the idiomatic way to bound a computed
quantity from both sides in one statement.

This is the same shape as a **least-cost input-output model** — rename FOOD→INPUT, NUTR→OUTPUT,
and the identical structure covers blending (crude/coal/animal-feed blending, where outputs are
blend qualities like octane or sodium content), production economics (inputs are activities,
outputs are products/demands — related to economic equilibrium, see
`../modeling-patterns/complementarity-problems.md`), and workforce scheduling (inputs are shift
schedules, outputs are day-by-day staffing requirements).

```ampl
set INPUT; set OUTPUT;
param cost {INPUT} > 0;
param in_min {INPUT} >= 0;  param in_max {j in INPUT} >= in_min[j];
param out_min {OUTPUT} >= 0; param out_max {i in OUTPUT} >= out_min[i];
param io {OUTPUT,INPUT} >= 0;
var X {j in INPUT} >= in_min[j], <= in_max[j];
minimize Total_Cost: sum {j in INPUT} cost[j] * X[j];
subject to Outputs {i in OUTPUT}:
   out_min[i] <= sum {j in INPUT} io[i,j] * X[j] <= out_max[i];
```

Gotchas:

- This pattern assumes **linearity of cost and yield** — proportional cost per unit input, and
  proportional/additive contribution to each output. Physically this can fail (e.g. octane
  rating from blending lead is nonlinear) — see `nonlinear-programs.md` when it does.
- Over-constraining (tight bounds on many outputs at once) commonly produces an
  **infeasible problem**. Diagnose by displaying `Con.lb`, `Con.body`, `Con.ub` for the last
  point the solver found — the one(s) where `body` sits outside `[lb, ub]` tell you which
  requirement is unmet.
  ```
  ampl: display Diet.lb, Diet.body, Diet.ub;
  ```
- To require whole-unit purchases (integer packages), add `integer` to the variable
  declaration (`var Buy {j in FOOD} integer >= f_min[j], <= f_max[j];`) — but this needs a
  MIP-capable solver and can noticeably change the optimal mix, not just round it. See
  `integer-programming.md`.
- `let paramname[index] := value;` is the fast way to explore "what if" changes to data from the
  interactive prompt without re-editing the `.dat` file — e.g. `let n_max["NA"] := 50000;`
  followed by `solve;` again.

## Pattern 3: Transportation / assignment (minimum-cost flow between two sides)

Use when: shipping/assigning a homogeneous quantity from a set of origins (each with a fixed
supply) to a set of destinations (each with a fixed demand), minimizing total shipping/
assignment cost. This is the simplest minimum-cost-flow shape; for genuine multi-node network
topologies see `network-models.md`.

```ampl
set ORIG; set DEST;
param supply {ORIG} >= 0;
param demand {DEST} >= 0;
check: sum {i in ORIG} supply[i] = sum {j in DEST} demand[j];   # feasibility precondition
param cost {ORIG,DEST} >= 0;
var Trans {ORIG,DEST} >= 0;
minimize Total_Cost: sum {i in ORIG, j in DEST} cost[i,j] * Trans[i,j];
subject to Supply {i in ORIG}: sum {j in DEST} Trans[i,j] = supply[i];
subject to Demand {j in DEST}: sum {i in ORIG} Trans[i,j] = demand[j];
```

The `check:` statement is worth including on every transportation-shaped model: total supply
must equal total demand for the equality constraints to be simultaneously satisfiable, and
`check` catches a data mismatch immediately after `data` is read rather than as a mysterious
infeasibility later.

**Assignment problem** is the special case where every `supply[i] = demand[j] = 1` (one person
per office, one machine per job, ...) — reuse the exact same model, just set
`param supply default 1; param demand default 1;` and let `cost[i,j]` represent preference/rank
rather than a dollar cost. A classical integrality result guarantees the optimal solution is
automatically all 0/1 (no `integer` declaration needed) whenever every supply/demand/bound is
integer.

**Reinterpretations that reuse this exact model unchanged**: shipping of any measurable good;
assignment of people to offices/jobs/shifts (cost = preference rank); any "flow that isn't
converted" — the raw material stays the same substance from origin to destination, unlike the
production/blending patterns where inputs are transformed.

`sum {i in ORIG, j in DEST} ...` , `sum {j in DEST, i in ORIG} ...`, and nested
`sum {i in ORIG} sum {j in DEST} ...` are all equivalent — order the indices however reads best.

**Sparse origin-destination pairs**: if not every origin can ship to every destination, don't
index over the full `{ORIG,DEST}` cross product — declare a pairs set and index everything over
it instead (see `03-sets-and-indexing.md` for the full "compound sets and slicing" treatment):

```ampl
set LINKS within {ORIG,DEST};
param cost {LINKS} >= 0;
var Trans {LINKS} >= 0;
minimize Total_Cost: sum {(i,j) in LINKS} cost[i,j] * Trans[i,j];
subject to Supply {i in ORIG}: sum {(i,j) in LINKS} Trans[i,j] = supply[i];
subject to Demand {j in DEST}: sum {(i,j) in LINKS} Trans[i,j] = demand[j];
```

## Combining patterns to build larger models

Real applications rarely stay this small; the book's Chapter 4 recipe for scaling up is:
**replicate a small model over an extra set, then add constraints that tie the replicas
together.** Three concrete recipes:

### Multi-commodity (replicate transportation over a PROD set)

Add `PROD` as an extra index to `supply`, `demand`, `cost`, and `Trans`. If the commodities
don't interact, this is just N independent transportation problems — not actually a reason to
combine them into one model. The reason to combine them is a **shared-capacity constraint**
that ties commodities together, e.g. a total shipping-capacity limit per lane:

```ampl
param limit {ORIG,DEST} >= 0;
subject to Multi {i in ORIG, j in DEST}:
   sum {p in PROD} Trans[i,j,p] <= limit[i,j];
```

Without a tying constraint like this, prefer solving separate smaller models — it's faster and
easier to understand.

### Multi-period (replicate over a time index, link with inventory-balance constraints)

Add a time set (`1..T`, or better, an `ordered set` of period labels — see
`03-sets-and-indexing.md` for ordered-set functions like `prev`/`first`). The link between
periods is an **inventory balance constraint**: what's available (produced + carried in) must
equal what's disposed of (sold + carried out):

```ampl
var Make {PROD,1..T} >= 0;
var Inv  {PROD,0..T} >= 0;
var Sell {p in PROD, t in 1..T} >= 0, <= market[p,t];
param inv0 {PROD} >= 0;                    # initial inventory, given as data
subject to Init_Inv {p in PROD}: Inv[p,0] = inv0[p];
subject to Balance {p in PROD, t in 1..T}:
   Make[p,t] + Inv[p,t-1] = Sell[p,t] + Inv[p,t];
maximize Total_Profit:
   sum {p in PROD, t in 1..T} (revenue[p,t]*Sell[p,t]
      - prodcost[p]*Make[p,t] - invcost[p]*Inv[p,t]);
```

Modeling the "initial condition" as an equality constraint (`Inv[p,0] = inv0[p]`) rather than
hard-wiring it is intentionally a little wasteful-looking — AMPL substitutes the constant
automatically, and it keeps the model uniform and easy to modify.

### Production + transportation (merge two patterns via a shared variable)

When a production model's output at each origin *is* the transportation model's supply at that
origin, drop the intermediate `supply` parameter and substitute the production variable
directly:

```ampl
subject to Supply {i in ORIG, p in PROD}:
   sum {j in DEST} Trans[i,j,p] = Make[i,p];   # was "= supply[i,p]" before merging
```

This is the general merge recipe: identify the quantity that plays the same role in both
sub-models, then replace the parameter/constraint that fixed its value in one sub-model with a
direct reference to the variable that computes it in the other.

## When *not* to combine

Bigger is not automatically better. If replicated instances of a model don't actually interact
(no shared resource, no linking constraint), solving them separately is faster, easier to
debug, and easier to explain. Only merge when there's a genuine coupling constraint or shared
variable — that coupling is exactly what makes the larger LP necessary rather than convenient.

## See also

- Sets/indexing needed for sparse pairs, ordered time periods, and multi-dimensional indexing:
  `../core-language/03-sets-and-indexing.md`
- True multi-node network topologies (not just origin/destination): `network-models.md`
- Nonlinear yield/cost relationships: `nonlinear-programs.md`
- Integrality requirements (whole units, assignment 0/1, fixed-charge logic):
  `integer-programming.md`
