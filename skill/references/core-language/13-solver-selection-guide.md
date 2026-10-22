# Solver Selection Guide

Derived from: solvers-index.md

Source: `raw/solvers-index.md` (`https://dev.ampl.com/solvers/index.html`), current as
of 2026-08-17 — this file's capability matrix is now sourced from AMPL's own solver
capability tables rather than editorial guidance. The "Picking a starting point" section
below remains practitioner synthesis layered on top of that sourced matrix, not itself
drawn from the corpus — see `SKILL.md`'s "Known gaps" section. Treat this as a starting
point, not a substitute for checking a solver's own documentation on
`https://dev.ampl.com/solvers/index.html` for version-specific capability (AMPL is
solver-agnostic by design; the book/API docs deliberately don't prescribe a solver).

## How to set a solver

```python
ampl.option["solver"] = "highs"
ampl.solve()
```
```ampl
option solver highs;
solve;
```

Install a solver module via `amplpy.modules install <name>` (see
`../amplpy/getting-started-and-installation.md`). Full mechanics of switching
solvers/solver options: `08-commands-modifying-models-and-data.md`,
`../amplpy/core-workflow-best-practices.md`. For tuning a specific solver once chosen,
see `../solver-options/`.

## Linear-quadratic solvers, by problem class

| Solver | LP | MILP | QP | MIQP | Convex (MI)QCP | Non-convex (MI)QCP | Conic | (MI)NLP |
|---|---|---|---|---|---|---|---|---|
| Gurobi | ✅ [GPU] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| FICO Xpress | ✅ [GPU] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| IBM CPLEX | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ≈ (PL approx.) |
| COPT | ✅ [GPU] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (SOCP + exp. cones) | ≈ (local optimum) |
| MOSEK | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ (SOCP + exp. cones) | ≈ (PL approx.) |
| NVIDIA cuOpt | ✅ [GPU] | ✅ [GPU] | ❌ | ❌ | ❌ | ❌ | ❌ | ≈ (PL approx.) |
| HiGHS | ✅ [GPU] | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ≈ (PL approx.) |
| CBC | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ≈ (PL approx.) |
| SCIP | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| GCG | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

"[GPU]" = GPU-accelerated algorithms available (e.g. PDLP for LP/MILP). "≈ (PL approx.)"
means: for solvers not supporting (MI)NLP directly, piecewise-linear approximation is
applied automatically (see `../mp-library/expressions-and-modeling-guide.md`). Conic
support elsewhere in the "Conic" column is SOCP only (not exponential cones).

## Nonlinear solvers, by problem class

| Solver | NLP | MINLP | Global optimum |
|---|---|---|---|
| Knitro | ✅ | ✅ | ❌ |
| BARON | ✅ | ✅ | ✅ |
| LINDO Global | ✅ | ✅ | ✅ |
| RAPOSa | ✅ | ✅ | ✅ |
| LGO | ✅ | ❌ | ✅ |
| CONOPT | ✅ | ❌ | ❌ |
| LOQO | ✅ | ❌ | ❌ |
| MINOS | ✅ | ❌ | ❌ |
| SNOPT | ✅ | ❌ | ❌ |
| IPOPT | ✅ | ❌ | ❌ |
| BONMIN | ✅ | ✅ | ❌ |
| COUENNE | ✅ | ✅ | ✅ |

All nonlinear solvers above are reached through AMPL via the MP2NL meta-driver where
they don't have a native MP interface (see
`../mp-library/tools-troubleshooting-and-reformulation.md`). Global optimality: even
solvers that accept integer variables (Knitro, Bonmin) only guarantee a *local* optimum
on non-convex models unless listed as "Global optimum".

## Quick lookup by problem class

| Problem class | Solver(s) | Notes |
|---|---|---|
| LP / MIP (default starting point) | **HiGHS** (open-source) | Gurobi, CPLEX, COPT, Xpress (commercial) are faster on large/hard instances — reach for one of these if HiGHS is too slow or a license is already available |
| QP / MIQP | HiGHS (LP + QP only), CBC | or any of the commercial LP/MIP solvers above, which also handle QP |
| Convex QCP / SOCP / conic | **Mosek** or **COPT** (both support SOCP + exponential cones) | Gurobi, CPLEX, Xpress also handle convex QCP/SOCP (SOCP only, no exponential cones) |
| Non-convex QCP | Gurobi, Xpress, COPT, SCIP, GCG | |
| MINLP (mixed-integer nonlinear) | Gurobi (v12+), Xpress, SCIP, GCG, BARON, Bonmin, Couenne | |
| General NLP (continuous, smooth nonlinear) | **Knitro**, **Ipopt** (open-source) | CONOPT, SNOPT, MINOS, LGO, LOQO |
| Global optimization (proven global optimum, non-convex) | BARON, LINDO Global, RAPOSa, Couenne, LGO | |
| GPU-accelerated LP/MIP | NVIDIA **cuOpt** | or Gurobi/Xpress/COPT/HiGHS configured with GPU-acceleration options |

## Picking a starting point

- **If the problem is linear or mixed-integer linear and there's no existing license**,
  start with HiGHS — open-source, installs in one line via `amplpy.modules`, and a
  reasonable baseline before reaching for a commercial solver.
- **If a commercial license is already available** (Gurobi, CPLEX, etc.), prefer it once
  the model is large or slow to solve — commercial LP/MIP solvers generally outperform
  open-source ones at scale, and most support the same `option solver` /
  `<solver>_options` mechanism, so switching is usually a one-line change
  (`08-commands-modifying-models-and-data.md`).
- **Nonlinear problems (NLP/QCP/MINLP) are more solver-sensitive than LP/MIP** — a
  problem that fails or converges to a poor local optimum with one solver may succeed
  with another, since NLP solvers differ meaningfully in algorithm (interior-point vs.
  active-set), handling of nonconvexity, and starting-point sensitivity. If a nonlinear
  solve behaves unexpectedly, trying a second solver from the same row above is a
  reasonable diagnostic step before assuming the model itself is wrong — see also the
  "multiple local optima" and other NLP pitfalls in
  `../modeling-patterns/nonlinear-programs.md`.
- **"Global optimization" solvers** (BARON, Couenne, LGO, ...) trade speed for a
  certificate of global optimality on non-convex problems — reach for these specifically
  when a locally-optimal solution isn't good enough, not as a default for every
  non-convex model.

## See also

- Setting/switching solvers, solver-specific option strings:
  `08-commands-modifying-models-and-data.md`
- Presolve, `solve_result` codes, diagnosing a solve that didn't go as expected:
  `11-solver-interaction-and-suffixes.md`
- Installing solver modules via amplpy: `../amplpy/getting-started-and-installation.md`
- NLP-specific pitfalls (why a solver might fail or misbehave):
  `../modeling-patterns/nonlinear-programs.md`
- MIP-specific solver-tuning advice (gap tolerances, warm starts):
  `../modeling-patterns/integer-programming.md`
- Per-solver tuning (autotuner, MIP stopping criteria, branching, numerical issues, cuts,
  algorithm/concurrent selection): `../solver-options/`
- The MP solver-interface layer (expanded expressions, multi-objective, numerical
  accuracy, reformulation tooling): `../mp-library/`
- Core (non-solver-specific) AMPL options: `14-ampl-options-reference.md`
