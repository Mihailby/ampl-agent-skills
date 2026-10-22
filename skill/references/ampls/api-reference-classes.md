# ampls Python API Reference: AMPLModel, GenericCallback, Solver-Specific Classes

Derived from: ampls-reference.md, ampls-amplmodel.md, ampls-gurobimodel.md, ampls-cplexmodel.md

Source: `ampls-reference.md`, `ampls-amplmodel.md`, `ampls-gurobimodel.md`, `ampls-cplexmodel.md`.

Condensed method reference — see `callbacks-and-examples.md` for task-oriented worked
examples. Note some methods appear in the source docs in `camelCase` (older) and
`snake_case` (newer) forms across different examples — both exist in the actual API;
if one form errors, try the other, and check the installed package's own docstrings for
the authoritative current form for its version.

## Module layout

`ampls` core types (`AMPLModel`, `GenericCallback`, `CutDirection`, `CBWhere`, `Status`,
plus the `amplpy.AMPL` extension methods) are re-exported by every solver-specific
package (`amplpy_gurobi`, `amplpy_cplex`, `amplpy_copt`, ...) — as long as the same
`ampls` library version underlies them, code written against these shared types is
portable across whichever solver package is actually imported. Each solver package
additionally exports its own model/callback subclasses (`GurobiModel`/`GurobiCallback`,
`CPLEXModel`/`CPLEXCallback`, etc.) and raw wrappers to that solver's native C library
entry points.

## `AMPLModel` — the base, solver-agnostic model interface

Loaded via `SolverDriver.loadModel()` (C++) or `amplpy.AMPL.to_ampls(drivername)`
(Python). Holds the in-memory solver representation plus a two-way variable-name ↔
solver-column-index mapping.

| Method | Purpose |
|---|---|
| `getVarMap()` / `getVarMapInverse()` | AMPL variable name ↔ solver column index, both directions |
| `getVarMapFiltered(prefix)` | name→index map restricted to variables starting with `prefix` — avoids materializing a possibly-huge full map |
| `optimize()` | run the solve |
| `getStatus()` | solution status, as an `ampls.Status` enum value |
| `getObj()` | current objective value |
| `getNumVars()` | variable count |
| `getSolutionVector()` | all variable values, as a plain tuple/list (column order) |
| `getSolutionDict()` | `{AMPL variable name: value}` dictionary |
| `writeSol(filename)` | write an AMPL-compatible `.sol` file |
| `getFileName()` | the `.nl` file this model was loaded from |
| `setCallback(callback)` | attach a `GenericCallback` (or solver-specific) subclass instance |
| `enableLazyConstraints()` | must be called once, before solving, if the callback will call `addLazy`/`addLazyIndices` |
| `printModelVars(onlyNonZero)` | debug utility: dump variable values to screen |
| `error(code)` | human-readable message for a numeric error code |
| `setAMPLParameter(param, value)` / `getAMPLIntParameter(param)` / `getAMPLDoubleParameter(param)` | generic (solver-independent) parameter get/set, using the `ampls.SolverParams` enumeration — e.g. `model.setAMPLParameter(ampls.SolverParams.INT_LP_Algorithm, ampls.LPAlgorithms.Barrier)`. For controls not covered by this generic enumeration, drop to the solver-specific API. |

## `GenericCallback` — solver-independent mid-solve hook

Base class to subclass for solver-agnostic callback logic; implement `run()` and attach
via `AMPLModel.setCallback()` before `optimize()`.

| Method | Purpose |
|---|---|
| `run()` | **override this** — called periodically during the solve |
| `getWhere()` | raw, solver-specific "where in the solve" code (unmapped) |
| `getAMPLWhere()` | the same, mapped to the portable `ampls.CBWhere` enum (see below) — not every solver-specific "where" has a generic mapping; unmapped cases surface as `CBWhere.NOTMAPPED` |
| `getWhereString()` | textual description of the current callback stage |
| `getMessage()` | the text being printed, valid only when `getAMPLWhere() == CBWhere.MSG` |
| `getSolutionVector()` / `getSolutionDict()` | current (in-progress) solution — **not callable at every stage**; check which `CBWhere` values support it |
| `getObj()` | current objective value (also stage-dependent) |
| `getValue(v)` | a mapped solver value via the `ampls.Value` enumeration (e.g. iteration count, MIP gap) — not every value is accessible this way; solver-specific docs cover the rest |
| `getVarMap()` / `getVarMapInverse()` / `getVarMapInverse(name)` | same mapping as `AMPLModel`, available mid-callback |
| `addCut(vars, coeffs, direction, rhs)` / `addCutsIndices(nvars, vars, coeffs, direction, rhs)` | add a user cut, by variable name list or by solver-index list |
| `addLazy(vars, coeffs, direction, rhs)` / `addLazyIndices(...)` | add a lazy constraint (requires `enableLazyConstraints()` to have been called first) |

`direction` in the cut/lazy methods is an `ampls.CutDirection` value: `LE` or `GE`.
Returning `-1` from `run()` tells the solver to interrupt the solution process — the
standard mechanism for implementing custom stopping criteria (see
`callbacks-and-examples.md`).

## Shared enumerations

**`CutDirection`**: `LE` (≤), `GE` (≥).

**`CBWhere`** (generic callback stage, via `getAMPLWhere()`): `MSG` (solver wants to print
something — read via `getMessage()`), `PRESOLVE`, `LPSOLVE` (running simplex),
`MIPNODE` (exploring a MIP node), `MIPSOL` (found a new MIP solution), `MIP` (general MIP
algorithm phase), `NOTMAPPED` (solver-specific stage with no generic equivalent — consult
that solver's own callback docs).

**`Status`** (generic solve outcome, via `AMPLModel.getStatus()`): `UNKNOWN`, `OPTIMAL`,
`INFEASIBLE`, `UNBOUNDED`, `LIMIT_ITERATION`, `LIMIT_NODE`, `LIMIT_TIME`,
`LIMIT_SOLUTION`, `INTERRUPTED`, `NOTMAPPED`.

## `GurobiModel` / `GurobiCallback`

`GurobiModel` extends `AMPLModel`: `getIntAttr(name)` / `getDoubleAttr(name)` (scalar
attributes, by Gurobi's own C-library attribute name string), `getIntAttrVector(name)` /
`getDoubleAttrVector(name)` (array attributes), `getIntParam(name)` / `getDoubleParam(name)`
/ `setParam(name)` (Gurobi parameters, by native name), `getGRBmodel()` / `getGRBenv()`
(raw native pointers, for solver-native usage — `usage-and-quickstart.md`).

`GurobiCallback` extends `GenericCallback`: `getCBData()` (raw callback data, for native
Gurobi C calls), `getGRBModel()` (underlying model pointer), `terminate()` (stop the
solve — the Gurobi-specific equivalent of returning `-1` from a generic callback's
`run()`), `getInt(what)` / `getDouble(what)` / `getDoubleArray(what)` (attributes
addressed by Gurobi's own enumeration values, for functionality beyond the generic
`getValue()`).

## `CPLEXModel` / `CPLEXCallback`

`CPLEXModel` extends `AMPLModel`: `getCPXLP()` / `getCPXENV()` (native CPLEX LP-model and
environment pointers), `setParam(CPXPARAM, value)` / `getIntParam(CPXPARAM)` /
`getDoubleParam(CPXPARAM)` (CPLEX control parameters, addressed via CPLEX's own
`CPXPARAM_*` enumeration).

`CPLEXCallback` extends `GenericCallback`: `getInt(what)` / `getDouble(what)`, addressed
via CPLEX's `CPX_CALLBACK_*` enumeration values.

**CPLEX-specific caveat worth flagging prominently**: when using callbacks with CPLEX,
multithreading must be handled explicitly by the user — set `model.set_option("threads",
1)` (or manage threading yourself) rather than assuming CPLEX's default multithreaded
callback behavior "just works" the way it might with other solvers. This appears
repeatedly as a guard clause across the example scripts in `callbacks-and-examples.md`
(`if SOLVER == "cplex": model.set_option("threads", 1)`), which suggests it's a common,
easy-to-miss gotcha rather than an edge case.

## See also

- Conceptual model (three usage levels, class hierarchy, why each class exists):
  `overview-and-architecture.md`
- Complete worked callback scripts using every class/enum above: `callbacks-and-examples.md`
- The higher-level `amplpy.AMPL` methods (`to_ampls`, `import_ampls_solution`, `exportGurobiModel`,
  etc.) that bridge into this API: `overview-and-architecture.md` (Integration with amplpy)
