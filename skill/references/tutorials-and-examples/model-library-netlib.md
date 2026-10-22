# Netlib AMPL Model Library

Derived from: netlib-ampl-models.md

Source: `netlib-ampl-models.md` (index page of https://www.netlib.org/ampl/models/,
maintained by David M. Gay).

## What this is

Netlib's `ampl/models` directory is a long-standing (pre-`amplpy`-era) public archive of
example `.mod`/`.dat` files and small conversion utilities, referenced by the AMPL book
and various AMPL papers. Useful as a source of additional worked examples beyond this
skill's own reference material, or for locating the original data behind a book figure —
not something to distill into prose (it's a file index, not explanatory content), so this
file is a categorized pointer list rather than a narrative reference.

## Contents by category

**Complementarity problems**: `compl/` library — sample complementarity problems (see
`../modeling-patterns/complementarity-problems.md` for AMPL's `complements` syntax).

**CSTR production/distribution models** (referenced in the AMPL book's early chapters):
`dist.mod`/`dist.n` (network version) + `dist03/08/13.dat`; `prod.mod` +
`prod03/08/13.dat`; `mod8.mod`/`mod8.dat` (Figures 1-3/1-4 in the book); `egypt1.mod`/
`egypt2.mod` (ordered-pair vs. original formulations of the EGYPT model) + matching
`.dat` files; `oil.mod`/`oil.dat`.

**MPS-format conversion utilities**: `m2a`/`m2ai` (awk scripts converting MPS data to
AMPL `data` statements, for use with `mps.mod`/`mps1.mod`/`mpsi.mod` respectively — `i`
variants handle MPS files with integer `MARKER` lines). `mps.mod` is shorter but may
reorder rows; `mps1.mod` preserves original row order — pick based on whether row order
matters for your downstream use.

**Nonlinear models**: `nlmodels/` library, a nonlinear-model collection assembled by
Elena Bobrovnikova — a useful supplementary source of worked nonlinear examples beyond
`../modeling-patterns/nonlinear-programs.md`.

**Structural/specialized-format examples**: `p2gon.mod` / `pgon.mod` (maximize the area
of a unit-diameter N-gon — two versions by Prieto); `score.mod` + `score1.dat` and
`struc.mod` + `struc1/6/8.dat` (Appendices B and C of the paper "Expressing Special
Structure...", technical memo TM 11274-910530-06).

**Transportation/network**: `train.n` (node-and-arc formulation), `train1.mod`/
`train2.mod` + matching `.dat` files.

**Reference values**: `optimal.val` — known optimal objective values for a subset of the
models above, useful for verifying a solver/formulation reproduces the expected answer.

## See also

- AMPL's own `node`/`arc` network syntax that `dist.n`/`train.n` demonstrate:
  `../modeling-patterns/network-models.md`
- `complements` syntax for the `compl/` library's problems:
  `../modeling-patterns/complementarity-problems.md`
- Nonlinear modeling guidance for working with the `nlmodels/` examples:
  `../modeling-patterns/nonlinear-programs.md`
