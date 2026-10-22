# Multi-Objective Optimization with MP

Derived from: mp-multiple-objectives.md

Source: `mp-multiple-objectives.md` (`https://mp.ampl.com/modeling-mo.html`).

Many real-world models have more than one objective. The classic approaches — blending
objectives into one linear combination at model-formulation time, or minimizing deviation
from pre-specified goals — both work in plain AMPL. MP adds first-class support for
solving a model with **several declared objectives directly**, either blended or
hierarchical, natively where the solver supports it and emulated otherwise. (Soft
constraints modeled as penalty terms folded into an objective are a related but separate
technique — not covered here.)

## Turning on multi-objective solving

A model can declare more than one `minimize`/`maximize` objective:

```ampl
minimize total_cost {s in STORE}:
   sum {j in FOOD} cost[s,j] * Buy[j];

minimize total_number:  sum {j in FOOD} Buy[j];
```

By default, AMPL/MP only optimizes the *first* declared objective (or whichever one is
named via `obj:no`). To actually consider all of them, set the solver option `obj:multi`
(alias `multiobj`) — see `feature-and-solver-options-guide.md` for its full value table.
With `obj:multi` on, objectives are blended by default; hierarchical (lexicographic)
treatment is opt-in via the `.objpriority` suffix described below.

## Blended objectives

All declared objectives are summed (with their respective minimize/maximize signs) unless
told otherwise. Suffix `.objweight` scales individual objectives — and can flip an
objective's effective sense — controlled by option `obj:multi:weight`:

```ampl
suffix objweight IN;                       # Declare the suffix
let {s in STORE} total_cost.objweight := TCFactor[s];
let total_number.objweight := 1;
```

## Lexicographic (hierarchical) objectives

To optimize objectives in priority order instead of blending them, declare and set suffix
`.objpriority` (see the `obj:multi` option description for the exact hierarchical
semantics). Higher-priority objectives are optimized first; lower-priority ones are then
optimized without degrading the higher-priority results beyond the tolerances below.

```ampl
suffix objpriority IN, integer;
suffix objabstol IN;
suffix objreltol IN;

maximize ReverseSeniority {e in 1..2, i in I: E[i]==e}:
  sum {t in V[i]: Pr[i, t]==0}
    S[i] * x[i, t]
  suffix objpriority (2-e)*S_range + 1 + S[i] - min {j in I} S[j];
```

`.objabstol` and `.objreltol` allow a controlled amount of objective degradation when
moving to the next priority tier. Their exact meaning can vary by solver's native
multi-objective mode (`obj:multi=1`), particularly for LPs — check "Support by solvers" in
`feature-and-solver-options-guide.md` and the AMPL Colab notebooks tagged
`lexicographic-objectives` (`https://colab.ampl.com/tags/lexicographic-objectives.html`)
before relying on exact tolerance semantics for a specific solver.

**Objectives sharing the same priority are blended together** within that tier — priority
groups the objective list, it doesn't uniquely order every objective.

## Per-objective solver options

Declare a suffix named `option_<optionname>` to set a solver option's value per objective
rather than globally:

```ampl
suffix objpriority integer;
suffix option_timelimit;    # single-word option alias
suffix option_mipgap;

option gurobi_options "obj:multi=2 timelimit=60 mipgap=0.00001";

minimize total_cost {s in 1..3}:
   sum {j in FOOD} cost[s,j] * Buy[j] suffix objpriority s;
minimize total_number:  sum {j in FOOD} Buy[j];

let total_number.objpriority := 3;   # alternative way to set priority

for {s in 1..3} {
    let total_cost[s].option_timelimit := s*10;
    let total_cost[s].option_mipgap := s*0.01;
}
let total_number.option_timelimit := 30;
```

Per-objective options are supported in the multi-objective emulator (`obj:multi=2`) and,
since AMPL/Gurobi build 20251211, natively in Gurobi (`obj:multi=1`). Set
`obj:multi:options=0` to ignore `option_*` suffixes entirely. Gurobi's own
`obj:N:priority` options do **not** currently interoperate with per-objective option
suffixes — use `.objpriority` as shown above instead.

**Zero means "not set."** AMPL suffixes default to 0, which this mechanism interprets as
"use the option's default (or whatever was set globally)." To set a real-valued
per-objective option *to* 0, use a very small nonzero value instead (e.g.
`let _obj[1].option_mipgap := 1e-20;`); this workaround is not currently available for
integer-valued options.

**Blended-group conflicts.** When two objectives share a priority (and are therefore
blended), the driver rejects the configuration if they've been given *different* values
for the same option — every objective in a blended group must either share an identical
option value or leave it unset. In the example above, `total_cost[3]` and `total_number`
end up in the same blended group (same priority); there's no conflict because
`option_timelimit` is 30 for both, and `option_mipgap` is only set on `total_cost[3]`.

## See also

- `feature-and-solver-options-guide.md` — the `obj:multi` option's full value table and
  the "Multiple objectives" row of the solver support matrix
- AMPL Colab notebooks: multi-objective examples
  (`https://colab.ampl.com/tags/multi-objective.html`), multi-objective options
  (`https://colab.ampl.com/tags/multi-objective-options.html`), lexicographic objectives
  (`https://colab.ampl.com/tags/lexicographic-objectives.html`)
- `overview-and-when-to-use.md` — MP entry point and which solvers are MP-based
