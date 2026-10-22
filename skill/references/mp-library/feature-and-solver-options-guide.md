# Feature and Solver Options Guide for MP-Based Solvers

Derived from: mp-feature-guide.md

Source: `mp-feature-guide.md` (`https://mp.ampl.com/features-guide.html`).

MP defines a set of standard *solver features* — capabilities characterized by a set of
options, sometimes AMPL suffixes for input/output data, and a defined effect on the
solution process. Because these are defined once at the framework level
(`overview-and-when-to-use.md`), option names and semantics are consistent across every
MP-based solver that implements a given feature, even though not every solver implements
every feature (see "Support by solvers" below). **For actually tuning a specific solver**
(which algorithm to pick, autotuning, branching strategy, cut generation, and other
solver-specific knobs) **see `../solver-options/`** — this file covers the
cross-solver mechanics and the common features MP standardizes, not per-solver tuning
advice, to avoid duplicating that content here.

## Important features at a glance

| Feature | Key option(s) |
|---|---|
| Time / solution-count limit | `lim:time` (`timelimit`), `lim:soltimelimit` |
| MIP gap / LP tolerance | `mipgap`, `mipgapabs`, `lp:opttol` |
| Report final MIP gap | `mip:return_gap` |
| Continuous-solve method | `alg:method`/`mip:nodemethod` |
| Crossover after barrier | `crossover`/`bar:basis` |
| LP basis in/out | `alg:basis`, `mip:basis` |
| Solving strategy/emphasis | `solutiontarget`, `optimalitytarget`, `heuremphasis`, `mipfocus`/`mipemphasis`, `norelwork` |
| Numerical emphasis | `presolve`, `numericfocus`/`numericalemphasis`, `scale`, `intfocus`, `refineops` |
| Feasibility/integrality tolerance | `feastol`, `inttol` |
| Universal bound for logical-expression variables | `cvt:bigM` (see `efficiency-and-numerical-accuracy.md`) |
| Piecewise-linear handling | AMPL option `pl_linearize=0` recommended with MP solvers |
| Multiple solutions | `sol:stub`, `sol:count`, `sol:poollimit` |
| Multiple objectives | `obj:multi` (see `multi-objective-optimization.md`) |
| Plateau stop (stall-based MIP termination) | `mip:plateau:time`, `mip:plateau:abstol`, `mip:plateau:reltol`, `mip:plateau:warmup`, `mip:plateau:log`, plus gap-based variants below |

## How AMPL/MP solver options work

### `<solver>_options` and `mp_options`

Solver-specific options are set via `<solvername>_options` (or, if the executable has been
renamed, e.g. `gurobi25.exe`, the matching `gurobi25_options`):

```ampl
option solver gurobi;
option gurobi_options 'iis=1';         ## Tell Gurobi to find IIS
solve;
```

Options common to **all** MP solvers can instead be set once via `mp_options`:

```ampl
option mp_options 'lim:time=300';      ## Applies to every MP solver
```

`mp_options` is parsed *before* `<solvername>_options`, so it sets a baseline that the
solver-specific string can then override for that one solver.

From the command line, options can go through an environment variable or as trailing
arguments:

```
mp_options='outlev=1 tech:writeprob=model.mps' gurobi.exe model.nl   # Method 1
gurobi.exe model.nl outlev=1 tech:writesol=model.sol                  # Method 2
```

### Listing and querying options

`<solver> -=` lists every available option; `<solver> -=key` filters to options containing
`key`:

```
$ highs -=acc:
acc: Options:

acc:linrange (acc:linrng)
    Solver acceptance level for 'LinConRange', default 2
```

The full option list per solver is also published at AMPL Development
(`https://dev.ampl.com/solvers/index.html`).

Query an option's current (default or previously set) value with `?`:

```ampl
option mosek_options 'threads=?';
solve;
## MOSEK 10.0.43:   tech:threads = 0
```

### Option files and native options

An option file can be loaded via `tech:optionfile`:

```ampl
option cbc_options 'optionfile="options_experiment1.txt"';
```

Some solvers also accept **native** (solver's own, non-AMPL-standardized) option names,
via `tech:optionnative`, alongside the standard AMPL/MP name for the same setting:

```ampl
option gurobi_options 'alg:numericfocus 3';                    ## standard AMPL/MP name
option gurobi_options 'tech:optionnative "NumericFocus 3"';    ## native Gurobi name
gurobi.exe model.nl optnative="numericfocus 2" optnative="Seed 500"   # from command line
```

`tech:optionnativeread`/`tech:optionnativewrite` read/write native options from/to a file,
for solvers that support it.

## Solve result codes

`solve_result_num`/`solve_result` report the outcome of the last solve
(`../core-language/11-solver-interaction-and-suffixes.md` covers the mechanism itself);
MP standardizes the numeric ranges across solvers:

| Range | Meaning |
|---|---|
| 0–99 | solved: optimal (or feasible, for a satisfaction problem) |
| 100–199 | solved?: solution candidate returned but error likely (150 specifically = MP solution check failed, `sol:chk:fail`) |
| 200–299 | infeasible |
| 300–349 | unbounded, feasible solution returned |
| 350–399 | unbounded, no feasible solution returned |
| 400–449 | limit, feasible: stopped early (iterations, time, Ctrl-C, ...) |
| 450–469 | limit, ambiguous: problem is either infeasible or unbounded — disable dual reductions or run the IIS finder for a definitive answer |
| 470–499 | limit, no solution returned |
| 500–999 | failure, no solution returned (550 specifically = numeric issue, no feasible solution) |

Individual solvers add more specific codes within these ranges; list them with `<solver>
-!` from the command line (e.g. `shell "mosek -!";` from within AMPL) or look them up at
AMPL Development.

## Support by solvers (condensed)

Legend: ✅ native, ✓ emulated (MP provides the behavior even though the underlying solver
doesn't), ❌ not supported. *Reconstructed from a fragmented HTML table fetch — cross-check
any single cell against `https://mp.ampl.com/features-guide.html#support-by-solvers`
before relying on it for a specific solver decision.*

| Feature | Copt | CPLEX | Gurobi | Mosek | Xpress | CBC | GCG | HiGHS | SCIP |
|---|---|---|---|---|---|---|---|---|---|
| Feasibility relaxation | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| IIS | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Sensitivity analysis | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Kappa | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Multiple solutions | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Multiple objectives | ✅ | ✅ | ✅ | ✓ | ✅ | ✓ | ✓ | ✓ | ✓ |
| Warm start | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| Lazy constraints / user cuts | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Return MIP gap | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Return best dual bound | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Plateau stop | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Plateau stop incl. MIP gap progress | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |

The raw source's full table also covers model/solution export, output basis, unbounded
rays, fixed-model/basis-for-MIP, and variable priorities per solver — see the live page if
one of those specifically drives a solver choice.

## General features

Each entry below: what it's for, the controlling option, and one example. Option/suffix
names are standardized across MP solvers per the "consistency" goal in
`overview-and-when-to-use.md`.

- **Output level** (`outlev`) — how much solver log output to print; `0` (default,
  suppressed — only the terminal `solve_message` shows), `1` detailed log, `2`+
  solver-dependent verbosity. `option solver_msg 0;` suppresses even the `solve_message`
  (the banner still prints; redirect `solve > NUL;` to suppress that too).
  `option <solver>_options 'outlev=1';`
- **Model export** (`writeprob`) — export the model before solving, format inferred from
  the file extension: `option <solver>_options 'writeprob=/tmp/diet.lp';`. Set
  `<solver>_auxfiles rc` (or `auxfiles rc` before `write`) first to carry variable/
  constraint/objective names into the export. In AMPL itself, `expand ConName;` shows the
  expanded constraint, and `solexpand;` shows the full AMPL-presolved model as sent to the
  driver (`option presolve 0;` first to see it without AMPL's own presolve applied).
- **Report solution time** (`tech:timing`/`tech:reporttimes`) — `0` off (default), `1`
  reports `time`, `time_setup`, `time_solver` suffixes on the problem, `2` adds
  `time_read`, `time_conversion`, `time_output`. `option <solver>_options 'timing=1';`
- **Warm start** (`alg:start`) — use incoming variable values (and, for LP, dual values) to
  start the algorithm instead of from scratch. `0` no, `1`–`3` yes with varying basis
  interaction (`3` = default: use incoming basis too, if any). Applicability: LP and MIP.
- **Input/output basis** (`alg:basis`) — pass/return a simplex basis via `status` suffixes
  on variables and constraints; `3` (default) does both. Re-solving with a sent basis can
  cut solve time to near zero for a marginally changed model.
- **Feasibility relaxation** (`alg:feasrelax`) — find a feasible solution to an infeasible
  model by allowing bound/constraint violations, penalized in the objective. Penalty
  weights: macro options `alg:lbpen`/`alg:ubpen`/`alg:rhspen` (default 1 each) or
  per-entity suffixes `lbpen`/`ubpen`/`rhspen` (default 0, meaning "use the macro
  default"); a negative penalty means "never violate this one." Values `1`–`3` choose the
  objective (weighted sum / weighted sum of squares / weighted count of violations), `4`–`6`
  add "and still optimize the original objective, subject to the violation being
  minimized." Applicability: LP and MIP.

  ```ampl
  option <solver>_options "alg:feasrelax=1"; option presolve 0;
  solve; display x, y;
  ```
- **Multiple solutions** (`sol:stub`, `sol:count`, `sol:poollimit`) — enumerate alternative
  (sub-optimal, for MIP) solutions found during search. Setting `sol:stub` auto-enables
  counting; solutions land in `<stub>1.sol` ... `<stub><nsol>.sol`, with `nsol`/`npool`
  reported as problem suffixes. `sol:poollimit` caps how many are kept.
- **Sensitivity analysis** (`alg:sens`) — ranges over which the optimal basis stays optimal.
  Output suffixes: `sensobjlo`/`sensobjhi`/`sensobj` (variables), and
  `senslblo`/`senslbhi`/`sensublo`/`sensubhi` (variables and constraints). LP and MIP.
- **Kappa** (`alg:kappa`) — condition number of the current LP basis matrix (stability
  measure); only available for a basic solution (barrier without crossover won't have
  one). `1` reports it in `solve_message`, `2` returns it in suffix `kappa` (on objective
  and problem).
- **Unbounded rays** (`alg:rays`) — for unbounded/infeasible linear models, return the
  direction of unboundedness (`unbdd` suffix on variables) or the infeasibility
  certificate (`dunbdd` suffix on constraints); `3` (default) returns both.
- **Multiple objectives** (`obj:multi`) — `0` off (default, only first/`obj:no` objective
  considered), `1` native if available else emulated, `2` always emulated. Full treatment:
  `multi-objective-optimization.md`.
- **IIS** (`alg:iisfind`/`iisfind`/`iis`) — for an infeasible model, find an Irreducible
  Inconsistent Subset: a minimal set of constraints/bounds that's still infeasible.
  `1` on; output is suffix `iis` (value `mem` for members) on variables and constraints.

## MIP-only features

- **Return MIP gap** (`mip:return_gap`) — `absgap = |objective - bestbound|`, `relmipgap =
  absgap / |objective|`. Sum of: `1` return `.relmipgap`, `2` return `.absmipgap`, `4`
  suppress gap text in `solve_message`. Gap suffixes report `+Infinity` if no
  integer-feasible solution exists yet.
- **Return best dual bound** (`mip:bestbound`) — `1` returns suffix `bestbound` on the
  objective (the best proven bound; `-Infinity`/`+Infinity` for min/max if unavailable).
- **Lazy constraints and user cuts** (`mip:lazy`) — reads suffix `lazy` on constraints:
  positive marks a lazy constraint (pulled in and enforced only once a candidate solution
  violates it — can cut off integer-feasible points), negative marks a user cut (pulled in
  to strengthen the relaxation, but redundant for integer feasibility). `3` (default)
  accepts both signs.
- **Variable priorities** (`mip:priorities`) — reads suffix `priority` on variables to
  guide branching order. `1` (default) read them, `0` ignore.
- **Fixed model / basis for MIP** (`mip:basis`) — after solving a MIP, fix all
  integer/SOS-constrained variables at their optimal values and resolve the continuous
  relaxation to obtain a basis, duals, and (if requested) sensitivity ranges that aren't
  otherwise available for MIP. `1` on; outputs suffixes `dual`, `sstatus` on variables.
- **Plateau stop** (`mip:plateau:time`, `mip:plateau:abstol`, `mip:plateau:reltol`,
  `mip:plateau:warmup`, `mip:plateau:log`) — stop the MIP search once the incumbent has
  stopped improving by at least `abstol` (absolute) or `reltol` (relative) for
  `mip:plateau:time` seconds, after an initial `mip:plateau:warmup`-second grace period.
  `mip:plateau:time=0` (default) disables the feature entirely. **`abstol`/`reltol`
  combine with OR, not AND** — since `abstol` defaults to 0 ("any change counts"), setting
  only `reltol` has no effect until `abstol` is also raised above the noise floor you want
  to ignore, because any nonzero absolute change already satisfies the OR.

  ```ampl
  option <solver>_options "mip:plateau:time=30 mip:plateau:abstol=1e-6 mip:plateau:reltol=0.001";
  solve;
  ```

  Stops if the incumbent hasn't improved by ≥0.1% (or, failing that, ≥1e-6 absolute) for
  30 seconds. `mip:plateau:log=1` prints a per-callback status line
  (`current=... previous=... progress=... elapsed=...`).

  For solvers that also report live MIP gap from the native callback,
  `mip:plateau:absgaptol`/`mip:plateau:relgaptol` add a second, independent progress
  channel — the shared timer resets if *either* the incumbent or the gap makes sufficient
  progress (opt-in: both default to 0, meaning that channel isn't tracked at all, unlike
  `abstol`/`reltol`'s "0 means any change counts"). The matching
  `mip:plateau:warmup:relgap`/`mip:plateau:warmup:absgap` let the warmup grace period end
  early once the reported gap first crosses one of these thresholds, instead of waiting
  out the full time-based `mip:plateau:warmup`. All four options, and the feature itself,
  only apply to solvers that support gap-based plateau stopping (see the support table
  above).

## See also

- `overview-and-when-to-use.md` — MP entry point, which solvers are MP-based
- `multi-objective-optimization.md` — full `obj:multi` treatment (blending, priorities,
  per-objective option suffixes)
- `efficiency-and-numerical-accuracy.md` — `cvt:bigM` and the numerical-focus options in
  their formulation context
- `tools-troubleshooting-and-reformulation.md` — `sol:chk:mode`/`chk:mode` and other
  solution-checking options referenced above
- `../core-language/11-solver-interaction-and-suffixes.md` — `solve_result`/
  `solve_result_num` mechanics outside the MP-specific numeric-range standardization
- `../solver-options/` — curated, per-solver option guides for actual tuning (algorithm
  choice, autotuning, branching, cuts) once a solver is chosen; this file intentionally
  does not duplicate that content
