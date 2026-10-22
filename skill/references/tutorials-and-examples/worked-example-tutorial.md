# Worked Tutorial: Lemonade Stand → Production Model

Derived from: basic-modeling.md, basic-commands.md, basic-scripting.md, modeling-and-data.md, commands-and-data.md

Source: `basic-modeling.md`, `basic-commands.md`, `basic-scripting.md`, `modeling-and-data.md`,
`commands-and-data.md` (AMPL's official "Getting Started" tutorial series, dev.ampl.com).

This is a narrative walkthrough of a single running example, progressively generalized —
useful as an end-to-end onboarding path, or as a source of copy-pasteable scaffolding
scripts (warm-start, multi-instance solving, generic solution printing). Individual
syntax elements introduced along the way are covered in more depth and more formally in
`../core-language/`; this file cross-references rather than re-explains them, and focuses
on the *pattern* each stage of the tutorial illustrates.

## Table of contents

- Stage 1: a concrete, non-parametric model (lemonade stand)
- Stage 2: solving and displaying, interactively and via API
- Stage 3: a second concrete instance (mulled wine) + warm-starting a MIP from its LP relaxation
- Stage 4: generalizing to a parametric model (production model) + `.dat` files
- Stage 5: solving multiple instances of the same abstract model
- Reusable script patterns worth keeping

## Stage 1: a concrete model

```ampl
var lemonade integer, >= 0;
var iced_tea integer, >= 0;
param fee = 2;
maximize profit: 1.5 * lemonade + iced_tea - fee;
subject to lemon_constraint: lemonade <= 10;
subject to tea_bag_constraint: iced_tea <= 8;
subject to sugar_constraint: 2 * lemonade + iced_tea <= 20;
```

A maximum-profit production model (two products, three resource constraints) — the
smallest possible complete AMPL model, useful as a first target before adding sets/indexing.
Full declaration syntax for `var`/`param`/`maximize`/`subject to`:
`../core-language/01-language-basics.md` through `05-variables-objectives-constraints.md`.

## Stage 2: solving and displaying

```
ampl: model lemonade.mod;
ampl: option solver highs;
ampl: solve;
ampl: display lemonade, iced_tea, profit;
```

- Prefer `model filename` over bare `include filename` for loading a model file — it
  guarantees AMPL is in model mode first (`../core-language/10-scripting-and-control-flow.md`
  covers `include` vs. `commands`; the same "prefer the mode-guaranteeing form" logic
  applies here to `model`/`data` vs. plain `include`).
- `option solver highs; solve;` — AMPL errors immediately and clearly
  (`No solver specified: option solver is ''`) if you `solve;` before setting a solver;
  don't be alarmed by this, it's not model-related.
- `highs_options "outlev=1"` (or `<solver>_options "..."`) — the solver-specific option
  string mechanism; see `../core-language/08-commands-modifying-models-and-data.md` for
  the general options mechanism and `../core-language/12-syntax-quick-reference.md` for
  command-line/invocation-level option persistence.
- `solve > highs.out;` redirects solver output to a file — useful for keeping a verbose
  `outlev=1` trace without cluttering interactive output.
- `show;` (no argument) lists every declared component by category; `show profit;`
  recalls one component's exact declaration — a fast way to check a model's current
  state without re-opening the `.mod` file. Full `show` argument-matching rules
  (`p` vs `pr` vs `pmt` as regex-like category prefixes) are in
  `../core-language/09-display-and-output-commands.md`.
- `print`/`printf` for one-off computed values (e.g. sales tax: `print "Sales tax", 0.0725*profit;`);
  `display` for anything indexed or set-shaped. This division of labor is covered in
  full in `../core-language/09-display-and-output-commands.md`.

## Stage 3: a second instance, and warm-starting

A second, structurally-identical model (`mulled_wine.mod` — same shape, different
products/ingredients/numbers) motivates two techniques:

**Rounding a relaxed LP solution as a MIP warm start**, using the generic `_var`/`_varname`
synonym arrays (`../core-language/09-display-and-output-commands.md`,
`../core-language/10-scripting-and-control-flow.md`):

```ampl
option relax_integrality 1; option solver highs; solve;      # solve the LP relaxation first
for {i in 1.._nvars} {
   if _var[i] != round(_var[i]) then { let _var[i] := round(_var[i]); }
}
option relax_integrality 0; option solver cbc; solve;         # now solve the real MIP, warm-started
```

The idea: solving the LP relaxation is cheap and gives a solution close to (though not
identical to) the eventual integer optimum; rounding it and feeding it back in as each
variable's *current value* before the real MIP solve gives the MIP solver a plausible
starting point rather than the default all-zero guess. This is a lightweight, generally
applicable warm-start technique whenever a good near-integer starting point is easy to get
cheaply — see `../modeling-patterns/integer-programming.md` for when warm starts matter
most (large/hard MIPs) and `../core-language/11-solver-interaction-and-suffixes.md` for
the underlying `reset_initial_guesses` mechanism that governs whether a variable's current
value is actually sent to the solver as a starting guess.

**A reusable "print the solution generically" script**, independent of which specific
model/variables are loaded, using only the `_var`/`_varname`/`_obj`/`_objname` generic
synonym families:

```ampl
printf "Solution with: %s\n", $solver;
for {i in 1.._nvars} { print _varname[i], "=", _var[i]; }
printf "%s = $%.2f\n", _objname[1], _obj[1];
```

Worth keeping as a standing utility script (`print_solution.run`) alongside any project
with more than one instance/model to solve, since it needs no editing when the model
changes — the whole point of using the generic synonym arrays instead of hard-coded
variable names.

## Stage 4: generalizing to a parametric model

The lemonade and mulled-wine models are the *same* production-model shape with different
numbers. Generalizing replaces hard-coded values with sets and indexed parameters:

```ampl
set PROD; set INGR;
var make {PROD} integer, >= 0;
param fee;
param profit_per_product {PROD};
param limit {INGR};
param usage {INGR, PROD};
maximize total_profit: sum {i in PROD} profit_per_product[i] * make[i] - fee;
subject to ingredient_constraints {i in INGR}:
   sum {j in PROD} usage[i,j] * make[j] <= limit[i];
```

This model declares no set members or parameter values at all — deliberately incomplete
until paired with a `.dat` file, which is the entire point: one abstract model, many data
instances. Sets/indexing fundamentals: `../core-language/03-sets-and-indexing.md`;
`.dat` file list/table formats: `../core-language/06-data-files-and-formats.md`.

```
ampl: model production.mod
ampl: option solver highs;
ampl: solve;
Error executing "solve" command: error processing var make[...]: no data for set PROD
```

This specific error message — "no data for set X" at `solve` time, not at `model` load
time — is the expected, correct behavior for an abstract model with no data loaded yet;
it's not a sign anything is wrong with the model itself.

## Stage 5: solving multiple instances of the same abstract model

```ampl
model production.mod;
data lemonade.dat;
include cbc_solve.run
include print_solution.run

reset data;                 # forget lemonade's data, keep the model
data mulled_wine.dat;
include highs_solve.run
include print_solution.run
```

`reset data;` is the key command here — it clears data-mode assignments only, leaving the
model declarations intact, which is exactly what's needed to re-instantiate the same
abstract model against a new data set without redeclaring anything. (Full `reset`/`reset
data`/`reset options` distinctions: `../core-language/08-commands-modifying-models-and-data.md`.)
This "load model once, loop `data` + `reset data` per instance" pattern is the natural
AMPL-native equivalent of the amplpy `for data, solver in zip(...): ... ampl.eval("reset
data;")` loop shown in `../amplpy/core-workflow-best-practices.md` — same idea, either
side of the API boundary.

**`reset;` vs. `reset data;` — pick based on what's actually changing.** The distinction
matters and is easy to blur: `reset data;` (above) is for re-running the *same* abstract
model against a *different* data instance. When instead you're switching to a genuinely
different, unrelated model file (e.g. going from the concrete `lemonade.mod` to the
concrete `mulled_wine.mod` — different variables, different constraints entirely), use
plain `reset;`, which clears both model and data so the next `model` command starts from
a clean slate:

```ampl
model lemonade.mod;
include cbc_solve.run
include print_solution.run

reset;                      # different model coming next -- clear model AND data
model mulled_wine.mod;
include cbc_solve.run
include print_solution.run
```

The amplpy equivalent is `ampl.reset()` (full reset) called between iterations of a loop
over *different* model files — as opposed to `ampl.eval("reset data;")` when looping over
data instances of the *same* model. See "Batch-solving multiple models/solvers" in
`../amplpy/core-workflow-best-practices.md` for the Python-side pattern, including a
reusable `set_n_solve(ampl, solver_name)` helper that keeps solver selection out of the
main driver loop — the amplpy equivalent of the `<solver>_solve.run` scripts above.

## Reusable script patterns worth keeping around

- **`print_solution.run`**: generic, model-independent solution printer (Stage 3 above).
- **`<solver>_solve.run`** (e.g. `highs_solve.run`, `cbc_solve.run`): a one-line script
  setting `option solver ...;` then `solve;` — keeps solver choice out of the main driver
  script, so swapping solvers is a one-line edit in one place.
- **Multi-instance driver** (Stage 5): `model` once, then `data`/`reset data;`/`solve`/print
  per instance — the standard shape for batch-solving several data sets against one model,
  whether driven from pure AMPL or from Python/R via amplpy (`../amplpy/core-workflow-best-practices.md`).

## See also

- Full formal syntax for every construct introduced here: `../core-language/` (see
  individual cross-references above)
- Cross-language equivalents (interactive AMPL vs. Python vs. R vs. C++ vs. C# vs. Java)
  for the API calls used throughout this tutorial: `cross-language-api-equivalents.md`
- amplpy-specific equivalents of the multi-instance and data-loading patterns shown here:
  `../amplpy/core-workflow-best-practices.md`
