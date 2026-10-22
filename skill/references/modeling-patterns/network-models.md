# Network Models: Transshipment, Max-Flow, Shortest Path, Assignment, node/arc

Derived from: ampl-book.md (ch. 15)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 15.

## Table of contents

- Why network structure matters (solvability, integer solutions, specialized algorithms)
- Algebraic (`var`/`subject to`) formulations: transshipment, max-flow, shortest path,
  transportation/assignment
- AMPL's native `node`/`arc` declarations: syntax, rules, worked conversions
- Side constraints and side variables (multicommodity flow, feedstocks)
- Keeping the "pure network" property; generalized networks

## The core idea: flow on arcs, balance at nodes

A **network linear program** restricts variables to represent flow along arcs, with only
two constraint types: bounds on arc flow, and conservation of flow at nodes ("flow in
equals flow out" or a generalization). This restricted structure is what earns network
models integer-optimal-solution guarantees (given integer data) and access to much
faster specialized solver algorithms — reason enough to prefer a network formulation
whenever a problem's structure permits one, and to be deliberate about *preserving* that
structure when extending a network model (see "Keeping the pure-network property" below).

## Algebraic formulation: general transshipment

```ampl
set CITIES;
set LINKS within (CITIES cross CITIES);
param supply {CITIES} >= 0;
param demand {CITIES} >= 0;
check: sum {i in CITIES} supply[i] = sum {j in CITIES} demand[j];   # flow-balance sanity check
param cost {LINKS} >= 0;
param capacity {LINKS} >= 0;
var Ship {(i,j) in LINKS} >= 0, <= capacity[i,j];

minimize Total_Cost: sum {(i,j) in LINKS} cost[i,j] * Ship[i,j];

subject to Balance {k in CITIES}:
   supply[k] + sum {(i,k) in LINKS} Ship[i,k] = demand[k] + sum {(k,j) in LINKS} Ship[k,j];
```

The `sum {(i,k) in LINKS} Ship[i,k]` idiom — summing over pairs where the *second*
component matches the outer dummy index `k` — is the standard way to express "flow into
node k" without a separate adjacency structure; the analogous sum with `k` first gives
flow out. This pattern recurs throughout network modeling (introduced for indexing in
general in `03-sets-and-indexing.md`, §6.2).

A single `net_demand` parameter (positive = warehouse, negative = plant, zero =
transshipment) is an equivalent, more compact alternative to separate `supply`/`demand`
when no city is simultaneously a source and a sink.

**Specializing for a known structure** (e.g. plant → distribution centers → warehouses,
never centers-to-centers) pays off in two ways: it makes illegal configurations
unrepresentable (declare `DW_LINKS within (D_CITY cross W_CITY)` rather than a generic
`LINKS`, so a center-to-center link simply cannot be entered as data), and it lets you
write balance constraints per city-type (`P_Bal`, `D_Bal`, `W_Bal`) instead of one generic
`Balance`. Trade-off: the generic version is easier to restructure later (change data
only); the specialized version is self-documenting and error-resistant but requires model
edits if the network's shape changes. A middle ground: define the specialized sets/params
in the model, then *derive* the generic `CITIES`/`LINKS`/`supply`/`demand` from them with
`=` expressions, keeping one shared constraint block:

```ampl
set CITIES = {p_city} union D_CITY union W_CITY;
set LINKS = ({p_city} cross D_CITY) union DW_LINKS;
param supply {k in CITIES} = if k = p_city then p_supply else 0;
```

### Common variations

- **Inequality balance**: replace `=` with `<=`/`>=` in a balance constraint when supply
  need not exactly match demand (e.g. plant can overproduce).
- **Loss in transit**: scale one side of the balance equation by a retention factor:
  `(1 - pd_loss[i]) * PD_Ship[i] = sum {...} DW_Ship[i,j];` — only a fraction of what's
  shipped arrives.
- **Unit conversion across a link**: multiply by a conversion factor
  (`(1000/ppc) * DW_Ship[i,j]`) when flow is measured in different units at each end
  (e.g. thousands of packages shipped vs. cartons received).

## Other classic network model classes (same variables/constraints, different objective)

- **Maximum flow**: drop the balance constraint at origin/destination, maximize flow out
  of the entrance (or into the exit): `maximize Entering_Traff: sum {(entr,j) in ROADS} Traff[entr,j];`,
  with `Balance` enforced only at intermediate nodes (`INTER diff {entr,exit}`).
- **Shortest path**: a max-flow-shaped model where supply/demand is exactly one unit;
  optimal arc variables come out 0/1, and the arcs with value 1 form the shortest/cheapest
  path. Add `subject to Start: sum {(entr,j) in ROADS} Use[entr,j] = 1;` and minimize
  total time/cost instead of maximizing flow.
- **Transportation / assignment**: the bipartite special case (every arc goes from an
  "origin" set straight to a "destination" set, no intermediate nodes) of the
  transshipment model. The same structure supports very different interpretations
  depending on what `supply`/`demand`/`cost` represent: hours of a person's time assigned
  to a project, a 0/1 people-to-projects matching, or a preference-ranking minimization —
  changing only the *meaning* of the data, not the model's shape.

## `node` and `arc`: declaring network structure directly

Algebraic constraints like `sum {(i,k) in LINKS} Ship[i,k] - sum {(k,j) in LINKS} Ship[k,j]`
obscure the network structure they represent — hard to verify by eye, and harder for
AMPL/solvers to recognize as a network for the purpose of applying specialized algorithms.
`node` and `arc` declarations let you state the network structure directly; AMPL derives
the equivalent variables and constraints automatically.

```ampl
minimize Total_Cost;                                    # no expression: obj phrases in arcs supply it
node Balance {k in CITIES}: net_in = demand[k] - supply[k];
arc Ship {(i,j) in LINKS} >= 0, <= capacity[i,j],
   from Balance[i], to Balance[j], obj Total_Cost cost[i,j];
```

- `node` replaces `subject to`; `arc` replaces `var`. `display Ship` still works — an arc
  name is usable as a variable name in commands (`09-display-and-output-commands.md`).
- `net_in` = flow in minus flow out at that node; `net_out` = flow out minus flow in
  (the negation). Pick whichever reads more naturally for the balance condition at hand.
- If a node has no explicit balance condition, `net_in = 0` (equivalently `net_out = 0`)
  is assumed — appropriate for pure transshipment nodes with no external supply/demand.
- `arc` bound phrases (`>=`, `<=`) work exactly like `var`; `from`/`to` name the connected
  nodes; `obj ObjName coeff-expr` supplies this arc's linear coefficient in the named
  objective, letting the `minimize`/`maximize` declaration itself be just a bare name (no
  `: expression`) — in which case the objective declaration must come *before* the arcs
  that feed it. If you'd rather write the objective algebraically using arc names as
  variables, declare the objective *after* the arcs instead.
- A `from`/`to` phrase can carry a trailing arithmetic factor (default `1`) to represent
  loss-in-transit or unit conversion — the `node`/`arc` equivalent of the algebraic
  variations above: `to Dist[i] 1-pd_loss[i]` or `from Dist[i] (ppc/1000)`.
- **`if`-conditioned `from`/`to`, not `if`-`then`-`else` on the node itself**: you cannot
  write `from (if i = entr then Entr_Int else Intersection[i])` — AMPL's if-then-else
  applies to *values*, not to model-component references, and this is rejected as a
  syntax error. Instead, give two `from`/`to` phrases, each with a `{if ...}` indexing
  guard, so exactly one applies per arc instance:
  ```ampl
  arc Traff {(i,j) in ROADS} >= 0, <= cap[i,j],
     from {if i = entr} Entr_Int,
     from {if i <> entr} Intersection[i],
     to   {if j = exit} Exit_Int,
     to   {if j <> exit} Intersection[j];
  ```
  Often cleaner still: fold the special-case nodes into the general one via a bound
  condition (`(if k=exit then -Infinity) <= net_out <= (if k=entr then Infinity)`), or
  treat entrance/exit as "dangling" arcs with only a `to` or only a `from` phrase
  (`arc Traff_In >= 0, to Intersection[entr];`) — three equivalent styles, pick whichever
  reads best for the network at hand.

### Side constraints and side variables

`arc`-declared components remain usable as ordinary variables in additional
`subject to` declarations ("side constraints"), and ordinary `var`-declared variables
can appear inside a `node`'s balance condition ("side variables") — both let you attach
non-network structure onto an otherwise pure network model without abandoning `node`/`arc`
for the network part itself:

```ampl
# Side constraint: joint capacity shared across products on the same physical link
subject to Multi {(i,j) in LINKS}:
   sum {p in PRODS} Ship[i,j,p] <= cap_joint[i,j];

# Side variable: a non-network variable feeding into a node's balance condition
var Feed {f in FEEDS, k in CITIES} >= 0, <= limit[f,k];
node Balance {p in PRODS, k in CITIES}:
   net_out = supply[p,k] - demand[p,k] + sum {f in FEEDS} yield[p,f] * Feed[f,k];
```

Both are ordinary multicommodity-flow-building-block patterns (replicate the network
over a `PRODS` set, then tie the copies together either via a shared-capacity side
constraint or via a shared-source side variable).

## Keeping the "pure network" property

A variable belongs to a **pure** network LP only if it appears in at most two
constraints (aside from its own bounds), with coefficient +1 in at most one and −1 in at
most one. This property is what guarantees all-integer optimal solutions (given integer
data) and unlocks specialized fast solver algorithms — worth deliberately preserving when
a model grows.

**Node-capacity example**: a naive throughput limit at each city —
`supply[k] + sum {(i,k) in LINKS} Ship[i,k] <= node_cap[k];` — is a *side constraint*
that adds a third coefficient per variable, breaking the pure-network property. The fix:
split each city into two nodes (`Supply[k]`, `Demand[k]`), route all links between the
`Demand` side of the origin and the `Supply` side of the destination, and represent
throughput as a bounded *arc* `Through[k]` from `Supply[k]` to `Demand[k]` instead of a
side constraint — same feasible region, but now expressed with only bounds and balance,
so the network structure (and its solver-recognizability) is preserved.

If your model uses only "simple" `node`/`arc` — no side variables in node conditions, no
loss/conversion factors on arcs — it is *guaranteed* to be a pure network LP. Reach for
`var`/`subject to` (or the side-variable/side-constraint patterns above) only when the
problem genuinely can't be expressed that way, and know that doing so is a deliberate
trade of solver speed/integrality guarantees for modeling flexibility.

**Generalized networks** (loss/conversion factors, i.e. coefficients other than exactly
±1) don't guarantee integer solutions but still admit fast specialized algorithms in
solvers that support the extension — check solver-specific documentation rather than
assuming.

## Solving: automatic network-algorithm detection

A model expressed via `node`/`arc` communicates its network structure to the solver
automatically; solvers with a network-simplex extension will report using it (e.g.
`"Network extractor found 7 nodes and 7 arcs. 7 network simplex iterations."`) without
any extra configuration. A model expressed algebraically (`var`/`subject to`) may or may
not be recognized as a network by the solver even when it genuinely has network
structure — some solvers need specific options set to detect it. This is one of the
strongest practical reasons to prefer `node`/`arc` over an equivalent algebraic
formulation whenever the network view is natural.

## See also

- The `sum {(i,k) in LINKS} ...` indexing idiom in general: `03-sets-and-indexing.md`
- Multi-period network models built by replicating a static model over time
  (inventory-as-arc pattern): `classic-lp-patterns.md`
- Multicommodity / side-constraint variants that break pure-network structure on purpose:
  this file's "Side constraints and side variables" section above
- Piecewise-linear costs on arcs (e.g. volume discounts on shipping): `piecewise-linear-programs.md`
