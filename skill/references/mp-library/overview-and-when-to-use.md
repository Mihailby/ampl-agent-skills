# MP Library: Overview and When to Use It

Derived from: mp-library.md, mp-modeling-overview.md

Source: `mp-library.md` (`https://mp.ampl.com/`), `mp-modeling-overview.md`
(`https://mp.ampl.com/modeling-overview.html`).

## What MP is

MP is AMPL's newer C++ solver-interface library — the recommended foundation for building
new AMPL solver drivers, publicly available in the `ampl/mp` GitHub repository
(`https://github.com/ampl/mp`). It provides type-safe, flexible interfaces suitable for
linear, mixed-integer, non-linear, and constraint-programming solvers, and is implemented
in C++ with parts of the API exported in C. Alongside the driver framework itself, MP
bundles a set of supporting tools: an automatic solution checker (built into every
MP-based solver, no configuration needed to get baseline checking), a Model Reformulation
Explorer, the MP2NL meta-driver, and a lightweight NL writer.

MP replaces the previous AMPL Solver Library (ASL) for solvers that don't require
automatic differentiation. In practice this means: if a solver driver is built on MP, it
inherits a materially larger vocabulary of AMPL expressions it can accept and solve than
an ASL-based driver would.

## Why MP exists

Three concrete benefits drive MP's design:

- **Automatic reformulation.** Problems containing expressions a solver doesn't natively
  support (conditional logic, counting operators, complementarity, and more — see
  `expressions-and-modeling-guide.md`) can still be solved through an MP-based driver: the
  interface transforms the expression into a form the underlying solver accepts, without
  the modeler having to hand-linearize it first.
- **Consistency.** Many solver options are implemented once at the framework level (see
  `feature-and-solver-options-guide.md`) rather than per solver, so an individual driver
  only needs to implement the corresponding API hook. This keeps option naming and
  semantics consistent across solvers — `outlev`, `lim:time`, `mipgap`, and similar options
  behave the same way regardless of which MP-based solver is active.
- **Speed of driver development.** Declarative, modern design patterns mean a typical MIP
  solver driver can be coded in a few days rather than a much longer bespoke integration
  effort — this matters less to modelers directly, but it's part of why the list of
  MP-based solvers below keeps growing.

## Which solvers use MP

Currently available MP-based solver drivers:

| Driver name | Solver |
|---|---|
| `gurobi` | Gurobi |
| `cplex` | IBM ILOG CPLEX |
| `copt` | Cardinal Optimizer |
| `xpress` | FICO Xpress |
| `mosek` | MOSEK |
| `baronmp` | BARON |
| `highs` | HiGHS (open-source) |
| `cuopt` | NVIDIA cuOpt (open-source) |
| `cbc` | CBC (open-source, enhanced interface) |
| `scip` | SCIP (open-source) |
| `gcg` | GCG (open-source) |

For every other AMPL solver (Knitro, Ipopt, CONOPT, BARON's non-MP build, the legacy
`gurobiasl`/`cplexasl`/`xpressasl` drivers, and so on), **MP2NL** acts as a meta-driver:
it translates MP-compatible syntax and features down to whatever that solver's own NL
interface accepts, so the expanded expression support and reformulation machinery are
available even for solvers that were never built against MP directly. See
`tools-troubleshooting-and-reformulation.md` for how to invoke MP2NL, and
`../core-language/13-solver-selection-guide.md` for the full solver-capability picture
this fits into.

Binaries for MP-based solvers are distributed through the AMPL Portal
(`https://portal.ampl.com`), individually or in bundles. Solver options and features are
documented in `feature-and-solver-options-guide.md` and, per solver, at AMPL Development
(`https://dev.ampl.com/solvers/index.html`).

## What MP adds: new operator categories

The expanded MP interface recognizes AMPL syntax in more general contexts than the
classic ASL-based interpretation did — most notably, allowing variables to appear inside
conditional, logical, and counting constructs that previously had to be modeled by hand.
The categories, each covered in full in `expressions-and-modeling-guide.md`:

| Category                            | Representative syntax                                                    |
| ----------------------------------- | ------------------------------------------------------------------------ |
| Conditional operators               | `if-then-else`; `==>`, `<==`, `<==>`                                     |
| Logical operators                   | `or`, `and`, `not`; `exists`, `forall`                                   |
| Piecewise-linear functions          | `abs`; `min`, `max`; `<<breakpoints; slopes>>`                           |
| Counting operators                  | `count`; `atmost`, `atleast`, `exactly`; `numberof`                      |
| Relational and comparison operators | `>(=)`, `<(=)`, `(!)=`; `alldiff`                                        |
| Complementarity operator            | `complements`                                                            |
| Nonlinear operators and functions   | `*`, `/`, `^`; `exp`, `log`; `sin`, `cos`, `tan`; `sinh`, `cosh`, `tanh` |
| Set membership operator             | `in`                                                                     |

These aren't new AMPL syntax — they're existing AMPL constructs that MP-based solver
interfaces can now accept in places (and combinations) that earlier ASL-based drivers
either rejected or required the modeler to reformulate manually.

**This updates/extends, rather than duplicates, the AMPL book's treatment of these
constructs** — per the source manifest's documented conflict note, where the book and MP
describe the same construct, MP is the more current source for anything
solver-interface-related (expression handling, piecewise-linear/logical/complementarity
operators, solver option naming), while the book remains authoritative for core language
syntax MP doesn't redefine (sets, params, `.dat` files, scripting). Specific supersession
notes are called out inline in the relevant `mp-library/` files below — e.g.
`expressions-and-modeling-guide.md`'s piecewise-linear section against
`../modeling-patterns/piecewise-linear-programs.md`.

## Where to go next

This file is the entry point for the `mp-library/` folder. From here:

- **Modeling with the expanded syntax** — every operator category above, with worked
  examples: `expressions-and-modeling-guide.md`
- **Multiple objectives** — blended and lexicographic (hierarchical) objectives,
  per-objective solver options: `multi-objective-optimization.md`
- **Getting solvers to perform well and reliably** — formulation choices that speed up
  solving, plus coefficient/bound scaling for numerical stability:
  `efficiency-and-numerical-accuracy.md`
- **Looking under the hood** — flat constraints vs. expression trees, the Reformulation
  Explorer, automatic solution checking, exporting the solver's model, the MP2NL
  meta-driver, and general troubleshooting steps:
  `tools-troubleshooting-and-reformulation.md`
- **Solver options and features mechanically** — how `<solver>_options`/`mp_options` work,
  solve result codes, and which solvers support which features (feasibility relaxation,
  IIS, warm start, multiple solutions, etc.): `feature-and-solver-options-guide.md`

## See also

- Solver capability matrix and picking a starting point:
  `../core-language/13-solver-selection-guide.md`
- Presolve, `solve_result` codes, and suffix mechanics in general (not MP-specific):
  `../core-language/11-solver-interaction-and-suffixes.md`
- Per-solver curated option guides for tuning a specific solver once chosen:
  `../solver-options/` (referenced here; consult it for solver-specific tuning rather than
  duplicating that content in this folder)
