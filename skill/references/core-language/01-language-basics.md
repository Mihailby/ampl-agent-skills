# AMPL Language Basics

Derived from: ampl-book.md (ch. 1, Appendix A.1), ampl-style-guide.md

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed. (Fourer, Gay,
Kernighan), Chapter 1 and Appendix A.1; `ampl-style-guide.md`. Canonical source per project
rules: the book.

## What AMPL is, and why the model/data split matters

AMPL is a declarative algebraic modeling language: you describe the structure of an
optimization problem (sets, parameters, variables, objective, constraints), and separately
supply the data that turns the abstract model into a solvable instance. The same `.mod` file
can be paired with any number of `.dat` files or programmatically-supplied data — this
separation is the single most important habit for keeping models maintainable as they grow
from a handful of variables to thousands.

A model is not one problem — it's a template for an infinite family of problems. Data turns a
model into a specific **instance**.

## Anatomy of a model

Every AMPL model is built from five kinds of declarations:

- **sets** — the index domains (products, time periods, locations, ...)
- **parameters** (`param`) — the given data, scalar or indexed over a set
- **variables** (`var`) — the unknowns a solver determines
- **an objective** (`maximize` / `minimize`) — one formula to optimize
- **constraints** (`subject to`) — restrictions the solution must satisfy

Minimal example (a two-variable LP, no sets at all):

```ampl
var XB;
var XC;
maximize Profit: 25 * XB + 30 * XC;
subject to Time: (1/200) * XB + (1/140) * XC <= 40;
subject to B_limit: 0 <= XB <= 6000;
subject to C_limit: 0 <= XC <= 4000;
```

```
ampl: model prod0.mod;
ampl: solve;
ampl: display XB, XC;
```

The generalized, indexed form of the same idea — this shape (set → indexed params →
indexed var → sum-based objective → sum-based constraint) is the template for the vast
majority of AMPL models:

```ampl
set P;
param a {j in P};        # tons per hour
param b;                 # hours available
param c {j in P};        # profit per ton
param u {j in P};        # market limit
var X {j in P};
maximize Total_Profit: sum {j in P} c[j] * X[j];
subject to Time: sum {j in P} (1/a[j]) * X[j] <= b;
subject to Limit {j in P}: 0 <= X[j] <= u[j];
```

Notes on the translation from algebra to AMPL:

- `Σ` becomes `sum`, `∈` becomes `in`, subscripts `a_j` become `a[j]`, multiplication needs an
  explicit `*`.
- `{j in P}` is an **indexing expression** — it appears in declarations ("for each j in P") and
  in `sum` ("add up ... for each j in P"). It is used everywhere the algebraic model says "for
  each ... in ...".
- Every named statement (`maximize Name: ...`, `subject to Name: ...`) requires a name; the name
  is arbitrary but mandatory, and becomes how you refer to that entity's solution values later
  (`Name.dual`, `expand Name`, etc.).
- Layout is free-form: statements end in `;`, can span multiple lines, and can appear in almost
  any order — but a component must be *declared* before it's *used*. Names are case-sensitive
  (`time`, `Time`, `TIME` are three different identifiers).
- Comments start with `#` and run to end of line.
- Bounds belong in the `var` declaration when possible (`var X {j in P} >= 0, <= market[j];`)
  rather than as a separate constraint — cheaper for the solver and clearer to read.

## Reading a model like a practitioner

Prefer long, meaningful names over terse mathematical ones once a model will be maintained by
more than one person (`PROD`/`rate`/`avail` over `P`/`a`/`b`). Add a unit comment to every
parameter and variable declaration. See `02-style-and-best-practices.md` for the full naming
and formatting conventions.

## Running a model interactively

```
ampl: model steel.mod;      # read the model
ampl: data steel.dat;       # read the data
ampl: solve;                # translate + hand off to solver + retrieve results
ampl: display Make;         # show solution values
ampl: reset;                # erase the current model/data so a new one can be loaded
ampl: quit;
```

`model` and `data` are separate commands specifically to encourage keeping model logic and data
values in separate files. Filenames are not otherwise restricted by AMPL — naming and
organization are up to you and your OS.

## Solution-related suffixes worth knowing from day one

After `solve`, every variable and constraint carries extra values accessible via `.suffix`
notation — the deep reference is `11-solver-interaction-and-suffixes.md`, but the ones you'll
reach for immediately:

| Suffix | Meaning |
|---|---|
| `Var.lb`, `Var.ub` | lower/upper bound in effect |
| `Var.rc` | reduced cost |
| `Con` (name alone) | the constraint's marginal/dual value |
| `Con.body` | current value of the constrained expression |
| `Con.slack` | difference between body and the nearer active bound |

```
ampl: display Make.lb, Make, Make.ub, Make.rc;
ampl: display Time;              # constraint name alone -> dual/marginal values
```

A dual (marginal) value tells you how much the objective would improve per unit of relaxing
that constraint — indispensable for "what's the bottleneck?" questions.

## Debugging a model as you write it

- **`expand`** shows what a symbolic declaration turned into once sets/data are substituted —
  the single best tool for "why is my model doing X":
  ```
  ampl: expand Total_Profit;
  ampl: expand Time;
  ampl: expand Make['coils'];   # what coefficients does this one variable have?
  ```
- **Syntax errors** are caught when the model is read, and point at the offending token with
  `>>> ... <<<` markers.
- **Undefined-name errors** (e.g., typo'd a variable name) are also caught at model-read time.
- **Data validity errors** (e.g., a `check` failing, or a value violating a declared bound) are
  caught when you `solve`, not when you read the data.
- Add `check` statements to a model liberally — catching bad data at load time is far cheaper
  than debugging a wrong-looking optimal solution. See the pattern in `03-sets-and-indexing.md`
  and `04-parameters-and-expressions.md`.
- A model that solves cleanly but gives an obviously-wrong answer (all zeros, wrong product
  favored) usually means the formulation, not the software, is at fault — re-derive the
  algebraic form and compare against `expand` output line by line.

## AMPL interfaces

The command-line/interactive interface (`ampl:` prompt) is what this whole skill assumes and
is available identically in every environment; it's also what you use for scripting
(`10-scripting-and-control-flow.md`) and for anything running "headless." Graphical interfaces
and language APIs (Python via `amplpy`, C++ via `ampls`, etc. — see `references/amplpy/` and
`references/ampls/`) sit on top of the same underlying language and commands; whatever you type
at `ampl:` is what those APIs send under the hood, so this reference material transfers
directly.

## Where to go next

- Reusable formulation patterns (production, blending/diet, transportation/assignment,
  multi-period, multi-commodity): `../modeling-patterns/classic-lp-patterns.md`
- Full syntax rules (lexical rules, reserved words, operator precedence table): `12-syntax-quick-reference.md`
- Naming, formatting, and structuring conventions: `02-style-and-best-practices.md`
- Sets in depth: `03-sets-and-indexing.md`
