# Changelog

## [2.0.2] - 2026-08-26

### Changed
- `raw/mp-expressions-supported.md` re-fetched: MP flags Set membership
  (`in set-expr`/`interval[...]`) as under rework, recommending disjunctive
  constraints (`||`) instead — new example added. See `raw/sources.md`.
- `references/mp-library/expressions-and-modeling-guide.md` updated to match.

## [2.0.1] - 2026-08-20

### Added
- One-line `Derived from: <raw filenames>` traceability note added at the top of all 44
  reference files, listing the raw/ source(s) each was distilled from.

## [2.0.0] - 2026-08-17

### Added
- New `references/mp-library/` subfolder (6 files): `overview-and-when-to-use.md`,
  `expressions-and-modeling-guide.md`, `multi-objective-optimization.md`,
  `efficiency-and-numerical-accuracy.md`, `tools-troubleshooting-and-reformulation.md`,
  `feature-and-solver-options-guide.md` — covers AMPL's MP solver-interface library
  (expanded expression syntax, multi-objective solving, numerical-accuracy guidance,
  reformulation/solution-check tooling, MP2NL).
- New `references/solver-options/` subfolder (5 files): curated, non-exhaustive per-solver
  tuning guides for Gurobi, Xpress, CPLEX, COPT, HiGHS — autotuner, MIP stopping criteria,
  variable/branch selection, numerical-issue tolerances, cuts, concurrent/algorithm
  selection, each pointing to its raw source and original URL for the full option list.
- New `references/core-language/14-ampl-options-reference.md` — core (non-solver-specific)
  AMPL `option` settings: display/formatting, presolve, solve, execution, file, library,
  misc.
- New `references/tutorials-and-examples/mp-library-reference-models.md` — four small
  runnable models (multi-objective diet, two infeasible models, n-Queens) used across the
  new mp-library/ and solver-options/ content.
- 20 new raw sources added to `raw/`: 10 MP library pages (`mp.ampl.com`) and 10 AMPL/solver
  options pages (`dev.ampl.com`) — see `raw/sources.md` for the full list.

### Changed
- `core-language/13-solver-selection-guide.md` rewritten to source its capability matrix
  from `raw/solvers-index.md` (`https://dev.ampl.com/solvers/index.html`) instead of
  purely editorial guidance; expanded with separate linear-quadratic and nonlinear-solver
  tables. Its "Picking a starting point" section remains editorial synthesis (see
  `SKILL.md`'s "Known gaps").
- `SKILL.md` reference map, "How to use this skill," and "Default recommendations"
  updated to route to the new `mp-library/` and `solver-options/` content; "Known gaps"
  updated accordingly.

### Fixed
- N/A

### Flagged (see `raw/sources.md` and `SKILL.md` "Known gaps" for detail)
- `raw/xpress-mp-options.md` is incomplete (fetch-tool capture limit) — covers `acc:`
  through `log:xprs_maxpagelines` only. `solver-options/xpress-options.md` substitutes
  ASL-interface option names for the missing categories, flagged inline. Not pursued
  further per explicit instruction — revisit if a browser-based fetch becomes available.
- `mp-library/feature-and-solver-options-guide.md`'s "support by solvers" table was
  reconstructed from a fragmented HTML-table fetch — cross-check individual cells against
  the live page before relying on one for a solver decision.

## [1.0.1] - 2026-08-16

### Changed
- Renamed skill from `ampl-modeling-language` to `ampl-skill` (SKILL.md and folder name).

## [1.0.0] - 2026-08-16

### Added
- SKILL.md and 31 reference files (core-language, modeling-patterns, amplpy, ampls, tutorials-and-examples, troubleshooting-and-faqs).
- Default recommendations (API-first, Python/amplpy default) and solver selection guide.

### Fixed
- Full review of basic-scripting.md; added missing content (CLI options table, filtered `let`, `reset` vs `reset data`, batch-solve pattern).
