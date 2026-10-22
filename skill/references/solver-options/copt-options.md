# COPT Options

Derived from: copt-mp-options.md

Source: `raw/copt-mp-options.md` (`https://dev.ampl.com/solvers/copt/options.html`),
current as of 2026-08-17. COPT only has an MP-interface driver (no separate ASL
variant), so this file is drawn from a single source (227 top-level options). **This is
a curated subset of commonly-tuned options** — the handful a practitioner actually
reaches for, not the full reference. See the raw source or the URL above for the
complete list.

## Setting options

```ampl
option solver copt;
option copt_options 'outlev=1 lim:time=300 mip:gap=0.01';
solve;
```

Option names take `prefix:name` form; legacy flat aliases (e.g. `timelim`, `mipgap`)
still work and are listed in parentheses in the raw reference. Both `key=value` and
`key value` syntax are accepted inside the string. General mechanics
(`(solver)_options` vs. `mp_options`, option files, querying values, native-option
pass-through): see `../mp-library/feature-and-solver-options-guide.md`.

## Verbosity

`tech:outlev` (`outlev`) — 0\*/1, output logging verbosity (default 0, no logging).

```ampl
option copt_options 'outlev=1';
```

## Autotuner

**Not available.** Unlike Gurobi, the COPT MP option reference has no `tune:`/`tech:tune*`
family or equivalent automatic parameter-search feature — nothing resembling
`tech:tunebase` was found in the raw source. Parameter selection for COPT is manual;
the categories below (MIP strategy levels, cut levels, algorithm choice) are the
practical substitute.

## MIP stopping criteria

| Option | Effect |
|---|---|
| `lim:time` (`timelim`) | Wall-clock time limit in seconds |
| `mip:gap` (`mipgap`) | Max. relative optimality gap to stop at (default 1e-4) |
| `mip:gapabs` (`mipgapabs`) | Max. absolute optimality gap to stop at (default 1e-6) |
| `lim:nodes` (`nodelim`) | Max. branch-and-bound nodes explored |
| `lim:soltime` (`soltimelim`) | Extra time limit that only starts counting once a feasible solution has been found — COPT's closest analog to a "good enough, stop soon" plateau rule |

No feasible-solution-count limit (Gurobi's `lim:sol`) was found in the COPT MP
reference.

```ampl
option copt_options 'lim:time=300 mip:gap=0.01 lim:soltime=60';
```

## Variable/branch selection

COPT does not expose a direct branching-variable-selection strategy knob or
branch-priority suffix comparable to Gurobi's `mip:varbranch`/`.priority` — none was
found in the raw reference. The closest lever is `mip:strongbranching`
(`strongbranching`): -1 automatic (default), 0 off, 1 fast, 2 normal, 3 aggressive,
controlling how much strong branching (looking ahead at candidate branch variables) the
solver does.

```ampl
option copt_options 'mip:strongbranching=2';
```

## Numerical issues

COPT has no single "numeric focus/emphasis" dial like Gurobi's `alg:numericfocus` —
none was found in the raw reference. Instead, tune the individual tolerances and
scaling directly: `alg:feastol` (`feastol`, default 1e-6, primal feasibility),
`alg:dualfeastol` (`dualfeastol`, default 1e-6, dual/reduced-cost tolerance),
`mip:intfeastol` (`intfeastol`/`inttol`, default 1e-6, integer feasibility), and
`pre:scale` (`scale`) — -1 automatic (default), 0 off, 1 on — for problems with
extreme coefficient ranges. `mip:repair` (`miprepair`/`repair`) extends the time limit
and attempts to repair a MIP solution that trips numerical checks.

```ampl
option copt_options 'alg:feastol=1e-8 pre:scale=1';
```

## Cuts

COPT exposes cut aggressiveness per B&B phase rather than one global dial plus
per-family overrides: `mip:cutlevel` (`cutlevel`) sets the general level, while
`mip:rootcutlevel` and `mip:treecutlevel` override it specifically at the root node
and in the rest of the tree. All three share the same scale: -1 automatic (default),
0 off, 1 fast, 2 normal, 3 aggressive. `mip:rootcutrounds`/`mip:nodecutrounds` cap how
many rounds of cut generation run at the root / per node (default -1, automatic).

```ampl
option copt_options 'mip:rootcutlevel=3 mip:treecutlevel=1';
```

## Concurrent and algorithm selection

`alg:method` (`method`/`lp:method`/`lpmethod`) selects the LP algorithm: -1 automatic
(default), 1 dual simplex, 2 barrier, 3 crossover, 4 concurrent (simplex and barrier
racing in parallel, first to finish wins), 5 automatic choice between simplex/barrier,
6 first-order PDLP method (GPU-capable). When `alg:method=4`,
`lp:concurrentmode` further controls how the concurrent run is split (including GPU
usage, alongside `alg:gpumode`/`alg:gpudevice`). `pre:solve` (`presolve`) controls
COPT's presolve: -1 automatic (default), 0 off, 1 fast, 2 normal, 3 aggressive —
turning it off is a useful diagnostic when a solve behaves unexpectedly. `pre:dualize`
lets COPT solve the dual formulation instead of the primal when that's advantageous.

```ampl
option copt_options 'alg:method=4 pre:solve=0';   # concurrent simplex+barrier, no presolve
```

## See also

- General solver-options mechanics (`(solver)_options` vs. `mp_options`, native
  options, option files): `../mp-library/feature-and-solver-options-guide.md`
- Choosing a solver in the first place: `../core-language/13-solver-selection-guide.md`
- MIP-specific modeling advice (warm starts, gap tolerances in context):
  `../modeling-patterns/integer-programming.md`
