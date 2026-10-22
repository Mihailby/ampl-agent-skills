# ampls: Overview and Architecture

Derived from: ampls-introduction.md, ampls-ampl-overview.md, ampls-architecture.md, ampls-structure.md

Source: `ampls-introduction.md`, `ampls-ampl-overview.md`, `ampls-architecture.md`,
`ampls-structure.md`.

## What ampls is, and when to reach for it

`ampls` is a lower-level, open-source layer beneath `amplpy` — a set of lightweight
interfaces between AMPL and individual solvers' own C APIs. Where `amplpy`
(`../amplpy/`) gives you a solver-agnostic Python interface to AMPL itself, `ampls` gives
you **direct access to the in-memory solver object** AMPL would otherwise hand off to
opaquely — needed specifically when a task requires either solver callbacks (custom code
that runs *during* the solve — lazy constraints, custom cuts, early termination) or
solver-native functionality that AMPL's own interface doesn't expose. If you don't need
mid-solve control or a solver-specific feature, stay in `amplpy` — reach for `ampls` when
you genuinely need to reach past AMPL's abstraction into the solver itself.

Core capabilities:
- Read an AMPL model instance from an `.nl` file (AMPL's solver-interchange format).
- Write a solution back out as a `.sol` file, importable back into AMPL.
- A choice of three usage levels (see "Three levels of usage" below), from fully
  solver-agnostic to fully solver-native.
- Works with existing AMPL licenses when used alongside the AMPL solver drivers.

Implementation: core written in C++, wrapped via SWIG for other target languages
(Python being the one covered in this source material). Documentation is split into a
language-agnostic conceptual layer (this file, `usage-and-quickstart.md`) and
language-specific reference material (`api-reference-classes.md` covers the Python API).

## Architecture: base classes + one implementation per solver

Three abstract base classes define the interface every solver plugin must implement:

| Base class | Responsibility |
|---|---|
| `SolverDriver` | constructs a model instance in solver memory (the entry point — "load this .nl file into solver X") |
| `AMPLModel` | holds a reference to the in-memory solver model; wraps common functionality (solve, get status/solution/objective, access variable/constraint counts, assign a callback) |
| `BaseCallback` (→ `GenericCallback` for solver-independent code) | represents a callback invoked during solving; exposes properties available during the solve and functions to add cuts/lazy constraints or terminate early |

Each solver (Gurobi, CPLEX, COPT, Xpress, ...) provides its own derived classes — e.g.
`GurobiDrv`, `GurobiModel`, `GurobiCallback` — implementing these three interfaces and
adding solver-specific extensions on top. **A model instance is tied to the solver it was
created for** — you cannot build a model against one solver's in-memory representation
and then hand it to a different solver's solve routine; switching solvers means starting
over from `SolverDriver.loadModel()` (or `AMPL.to_ampls(drivername)`) with the new
solver's driver.

## Three levels of usage

The same architecture supports three distinct styles of code, trading off portability
against access to solver-specific power — pick per task, not per project, since a single
codebase can mix all three where appropriate:

1. **Generic usage** — code written against only the base-class functionality
   (`AMPLModel`, `GenericCallback`) works unmodified across every supported solver. This
   is the natural default when the logic you're writing (e.g. "print the objective value
   after each new MIP solution") doesn't depend on solver internals.
2. **Solver-specific usage** — using a solver's derived-class extensions
   (`GurobiModel.getDoubleAttr(...)`, `CPLEXModel.getIntParam(...)`) ties the code to that
   one solver, in exchange for access to that solver's own attribute/parameter surface as
   exposed through `ampls`'s wrapper.
3. **Solver-native usage** — a further step: obtaining the *raw native pointer*
   (`getGRBmodel()`, `getCPXLP()`/`getCPXENV()`) and calling the solver's own C library
   functions directly, bypassing `ampls`'s wrapper entirely. Use when a capability isn't
   exposed by either the generic or solver-specific `ampls` layers at all.

In Python specifically, `amplpy-gurobi` and `amplpy-cplex` (and similarly for other
solvers) each bundle the shared `base` group (`AMPLModel`, `BaseCallback`/
`GenericCallback`) alongside their own solver-specific classes and raw wrappers to the
solver's native C library functions — this is what makes solver-native calls possible
without leaving Python.

## Integration with amplpy

If `amplpy` is installed alongside an `ampls` solver package, importing the `ampls` module
adds these methods directly onto `amplpy.AMPL`:

| Method | Purpose |
|---|---|
| `ampl.exportGurobiModel(options=None)` | export the current model as a `GurobiModel` |
| `ampl.exportCplexModel(options=None)` | export as a `CPLEXModel` |
| `ampl.exportModel(drivername, options=None)` | generic export; `drivername` is `"gurobi"` or `"cplex"` (or another installed solver) |
| `ampl.import_ampls_solution(model)` | import an `ampls` model's solution back into the AMPL session |

`options` accepts the same option strings you'd pass to the AMPL solver driver directly
(as listed by `<solver> -=`), e.g.
`ampl.exportGurobiModel(["outlev=1", "return_mipgap=5"])`. This import/export round trip
— export to `ampls` for solver-level control, then `import_ampls_solution` to bring
results back — is the standard pattern for combining `amplpy`'s convenience with `ampls`'s
solver access; see `callbacks-and-examples.md` for full worked examples.

## See also

- Installing `ampls` packages and the three usage-level code patterns, worked in C++/Python
  syntax: `usage-and-quickstart.md`
- Full method reference for `AMPLModel`, `GenericCallback`, `GurobiModel`, `CPLEXModel`,
  and the shared enumerations: `api-reference-classes.md`
- Complete worked callback examples (generic callbacks, stopping criteria, lazy
  constraints for TSP subtour elimination, multi-model workflows): `callbacks-and-examples.md`
- The higher-level, solver-agnostic Python interface this sits underneath: `../amplpy/`
