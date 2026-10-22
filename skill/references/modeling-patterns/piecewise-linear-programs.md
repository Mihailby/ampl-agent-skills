# Piecewise-Linear Programs

Derived from: ampl-book.md (ch. 17)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 17.

## Table of contents

- The `<<breakpoints; slopes>> arg` notation and why it exists
- Fixed vs. varying number of pieces
- Common 2-/3-piece idioms: soft constraints, infeasibility diagnosis, reversible
  activities (backordering)
- Rewriting `if`/`max`/`abs` as explicit piecewise-linear terms (and why it matters)
- Non-separable cases (min-max) that need a genuinely different reformulation
- Guidelines: convexity/concavity, when this stays a linear program vs. becomes integer

## The notation, and why AMPL has it

```ampl
<<breakpoint-list; slope-list>> pl-argument
```

A piecewise-linear cost is common wherever a per-unit rate changes at volume thresholds
(shipping rates, hours-worked penalties, tiered pricing). Modeling it with separate
variables per segment (`Trans1`, `Trans2`, `Trans3`, one per rate tier, each individually
bounded) works but pollutes every downstream constraint and display command with
segment-level detail you don't actually care about. AMPL's `<<...>>` syntax describes the
function directly against the *original* variable, and is the only form AMPL's solver
translation machinery recognizes as piecewise-linear (see "Detection" below) — `if`,
`max`, and `abs`-based equivalents look right on paper but are silently treated as opaque
nonlinear expressions instead.

```ampl
param rate1 {i in ORIG, j in DEST} >= 0;
param rate2 {i in ORIG, j in DEST} >= rate1[i,j];
param rate3 {i in ORIG, j in DEST} >= rate2[i,j];
param limit1 {i in ORIG, j in DEST} > 0;
param limit2 {i in ORIG, j in DEST} > limit1[i,j];

minimize Total_Cost:
   sum {i in ORIG, j in DEST}
      <<limit1[i,j], limit2[i,j]; rate1[i,j], rate2[i,j], rate3[i,j]>> Trans[i,j];
```

- Breakpoint list and slope list are separated by `;`; items within each by `,`.
- One more slope than breakpoints (first slope applies below the first breakpoint, last
  slope above the last).
- **Convention: the function is zero at zero**, unless you override with the two-argument
  form of `pl-argument`: `<<...>> (arg-expr, zero-expr)` states explicitly where the
  function evaluates to zero (see "Anchoring at a nonzero point" below). Forgetting this
  is a common source of silently-wrong piecewise-linear costs — a shifted function with
  the wrong zero point still solves and produces plausible-looking numbers.

## Varying number of pieces (indexed breakpoint/slope lists)

When the piece count itself varies by instance (a link might have 1, 2, or 12 rate
tiers), hand-enumerating `rate1`...`rate12`/`limit1`...`limit11` params is unworkable —
use an indexed item inside the breakpoint/slope list instead:

```ampl
param npiece {ORIG,DEST} integer >= 1;
param rate  {i in ORIG, j in DEST, p in 1..npiece[i,j]}   >= if p=1 then 0 else rate[i,j,p-1];
param limit {i in ORIG, j in DEST, p in 1..npiece[i,j]-1} >  if p=1 then 0 else limit[i,j,p-1];

minimize Total_Cost:
   sum {i in ORIG, j in DEST}
      <<{p in 1..npiece[i,j]-1} limit[i,j,p]; {p in 1..npiece[i,j]} rate[i,j,p]>> Trans[i,j];
```

The indexing expression before an item must range over an **ordered set**; it expands to
one list entry per member. `npiece[i,j] = 1` yields zero breakpoints and one slope — a
plain linear term, the degenerate (and perfectly legal) case. The recursive `param`
definitions (`rate[...,p] >= ... rate[...,p-1]`) are the same self-referencing-parameter
pattern covered in `04-parameters-and-expressions.md`, used here specifically to enforce
that rates are non-decreasing across pieces as part of the data declaration itself.

## Common 2-/3-piece idioms

### Soft constraints via penalty terms

Replace a hard bound with a variable plus a penalty for exceeding a preferred level:

```ampl
var Use {t in 1..T} >= 0, <= avail_max[t];
subject to Time {t in 1..T}: sum {p in PROD} (1/rate[p]) * Make[p,t] = Use[t];
maximize Total_Profit:
   ... - sum {t in 1..T} <<avail_min[t]; 0, time_penalty[t]>> Use[t];
```

Zero penalty up to `avail_min[t]`, then `time_penalty[t]` per unit above it. Optimal
solutions to this style of model tend to "stick" at the breakpoints — expect many
variables to land exactly at `avail_min[t]` rather than smoothly distributed above it.

### Diagnosing infeasibility by concentrating it

A hard-infeasible model scatters small violations across many variables in ways that are
hard to interpret from the raw solver output (e.g. negative inventory appearing as a
side effect rather than a flagged problem). Instead, extend a bound into a very steep but
finite penalty region so the solver is *allowed* to violate it, but only when genuinely
forced to, and only just enough:

```ampl
var Use {t in 1..T} >= 0;                                    # upper bound removed
maximize Total_Profit:
   ... - sum {t in 1..T} <<avail_min[t], avail_max[t]; 0, time_penalty[t], 100000>> Use[t];
```

The former hard upper bound `avail_max[t]` becomes a second breakpoint with an
enormous slope (`100000`) beyond it — the solution will report exactly how far past the
old limit it had to go, concentrated and readable, rather than an opaque "infeasible."
Apply the same idea from the other side (penalize falling short of a *minimum*
commitment) by giving the function a very negative slope below the threshold and using
the two-argument form to anchor it correctly (see next section) — both approaches turn a
binary infeasible/feasible result into a graduated diagnostic.

### Anchoring at a nonzero point

Recall the zero-at-zero default: a penalty that should be *positive* at zero and fall to
zero at some threshold needs explicit anchoring, or you'll get the wrong function shifted
by a constant:

```ampl
<<commit[p,t]; -100000, 0>> (Sell[p,t], commit[p,t])
```

reads as "this function of `Sell[p,t]` is zero at `commit[p,t]`," which is the correct
"penalize falling short of the commitment" shape. Omitting the second `zero-expr`
argument here would silently produce a function that's zero at `Sell[p,t] = 0` instead —
wrong shape, no error, plausible-looking numbers.

A linear term and a piecewise-linear term on the *same* variable combine automatically
into one piecewise-linear function (AMPL detects this) — so a variable can appear both
inside `<<...>>` and elsewhere in the same linear objective/constraint without needing
manual consolidation.

### Reversible activities (cost that penalizes both directions from zero)

When a variable is meaningful at both positive and negative values (e.g. an inventory
variable reinterpreted as allowing backorders when negative), a *naive* removal of its
`>= 0` bound combined with a single linear cost term produces a nonsensical result: cost
computed as `cost * value` decreases (looks like profit!) as the variable goes more
negative, so the solver will happily drive it to an extreme. The fix is a V-shaped
piecewise-linear cost, explicit about penalizing deviation in *either* direction from
zero:

```ampl
param invcost {PROD} >= 0;     # cost per unit of positive inventory
param backcost {PROD} >= 0;    # cost per unit of backorder (negative inventory)
maximize Total_Profit:
   sum {p in PROD, t in 1..T} (... - <<0; -backcost[p], invcost[p]>> Inv[p,t]) - ...;
```

This is a real, easy-to-miss modeling bug class: any time you relax a variable's sign
restriction to model "can go negative too," check whether the *cost* term needs to become
piecewise-linear (penalizing both directions) rather than staying a single linear
coefficient (which only discourages one direction and rewards the other). An "end effect"
can still appear near the model's final period (nothing penalizes backorders that would
be cleared *after* the horizon ends) — a boundary condition like
`subject to Final {p in PROD}: Inv[p,T] >= 0;` is a common, simple fix.

## Detection: why `if`/`max`/`abs` don't work as substitutes

```ampl
if Use[t] > avail_min[t] then time_penalty[t]*(Use[t]-avail_min[t]) else 0    # NOT detected
max(0, time_penalty[t] * (Use[t] - avail_min[t]))                             # NOT detected
time_penalty[t] * abs(Use[t] - avail_min[t])                                  # NOT detected
```

All three are mathematically piecewise-linear, but AMPL's solver-translation layer does
**not** recognize them as such — it will treat them as opaque nonlinear expressions and
you're unlikely to get a satisfactory (or even correct-shaped) solve. Only the explicit
`<<...>>` syntax is recognized and gets the efficient LP-based translation. Rewrite
`abs(x)`-style deviation penalties as `<<breakpoint; -coeff, coeff>> x` and `if`/`max`
threshold penalties as shown above. Scaling: multiplying a piecewise-linear function by a
constant is equivalent to scaling every slope individually — `c * <<b; s1,s2>> x` ==
`<<b; c*s1, c*s2>> x`.

## When it's NOT expressible with `<<...>>`: non-separable functions

`<<...>>` only describes a function of a **single** variable. A "minimize the maximum
cost across people" (min-max) objective is piecewise-linear in aggregate but not
separable — it can't be written as a sum of per-variable pieces, so `<<...>>` cannot
express it. The standard reformulation introduces an auxiliary variable and a family of
`>=` constraints instead:

```ampl
var M;
minimize Max_Cost: M;
subject to M_def {i in PEOPLE}: M >= sum {j in PROJECTS} cost[i,j] * Assign[i,j];
```

Because `M` is minimized, it's forced to equal the true maximum at optimality — a
generally-applicable "epigraph" trick for min-max objectives (for max-min, reverse to
`maximize` with `M <= ...` constraints). Recognize this pattern whenever an objective
reads as "minimize the worst of several linear quantities" — it needs this reformulation,
not `<<...>>` and not a bare `max{}` in the objective.

## Guidelines: when piecewise-linear optimization stays easy

- **Convex** (nondecreasing slopes) piecewise-linear function **minimized**, or
  **concave** (nonincreasing slopes) **maximized**, subject to otherwise-linear
  constraints: AMPL translates this to an equivalent LP automatically and transparently —
  this is the case for every example in this file (shipping rates that *increase* with
  volume, in particular, is what keeps the classic transportation example easy; rates
  that *decreased* with volume — economies of scale — would make it a concave-minimize
  problem instead, i.e. hard).
- Any other combination (convex maximized, concave minimized, or a piecewise-linear term
  appearing on the "wrong" side of a constraint's inequality relative to its
  convexity/concavity) requires **integer programming** under the hood — AMPL still
  handles the translation automatically at `solve` time, but the result is a
  fundamentally harder combinatorial problem, not a disguised LP. See
  `integer-programming.md`.
- Difficulty scales with the *number of pieces*, not just their pattern — beyond roughly
  a dozen pieces per term, consider approximating with a genuinely nonlinear function
  instead (`nonlinear-programs.md`).
- `option pl_linearize 0;` disables AMPL's automatic LP/IP translation, for solvers that
  accept piecewise-linear structure natively (specialized "nondifferentiable" or
  piecewise-linear-aware solvers) — check solver-specific docs; a generic NLP solver is
  unlikely to handle raw piecewise-linear terms reliably without that native support.
- A constraint-side (not objective-side) piecewise-linear term keeps its LP tractability
  only in narrower cases: convex on the `<=` side (equivalently, on the RHS of `>=`), or
  concave on the `>=` side (equivalently, RHS of `<=`) — any other placement forces
  integer programming.

## See also

- Recursive parameter definitions used for the "non-decreasing rate" data pattern:
  `04-parameters-and-expressions.md`
- Full formal grammar for `<<...>>` (breakpoint/slope list forms, `pl-argument` forms):
  `12-syntax-quick-reference.md`, section A.4.3
- What happens when AMPL falls back to integer programming for a hard piecewise-linear
  case: `integer-programming.md`
- Smooth nonlinear alternatives when a piecewise-linear approximation needs too many
  pieces: `nonlinear-programs.md`
