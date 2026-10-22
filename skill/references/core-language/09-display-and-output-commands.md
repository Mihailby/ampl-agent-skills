# Display and Output Commands

Derived from: ampl-book.md (ch. 12)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 12.

## Table of contents

- `display`: sets, parameters, variables, indexed expressions
- Formatting options (list vs. table, width, zero suppression)
- Numeric precision/rounding options (display-only vs. solution-changing)
- `print` and `printf`
- Suffixes for solution values: bounds, slack, dual values, reduced cost
- Model-introspection commands: `show`, `xref`, `expand`, generic `_var`/`_con`/`_obj` synonyms
- Output redirection, logging, message limits

## `display`: the default tool for looking at anything

```
ampl: display ORIG, DEST, LINKS;                 # sets
ampl: display T;                                  # scalar
ampl: display avail["reheat"], avail["roll"];      # specific indexed entries
ampl: display avail;                               # whole indexed collection -> list or table
ampl: display sin(1)^2 + cos(1)^2;                 # arbitrary expression
```

- `display` picks list format for small amounts of data, table format for larger, matching
  `.dat`-file conventions as closely as possible (`06-data-files-and-formats.md`) — you rarely
  need to think about this; see below to control it explicitly.
- For **3+ dimensional** entities, AMPL slices into a series of 2-D tables, picking whichever
  dimension to slice on minimizes the number of tables — it does *not* always slice on the
  first index. To force a slice on a specific dimension, index the whole `display` command
  explicitly:
  ```
  ampl: display {p in PROD}: {i in ORIG, j in DEST} Trans[i,j,p];   # one table per member of PROD
  ```
- **You cannot write "array arithmetic"** like `display revenue * Sell;` — AMPL has no
  element-wise operators outside an indexing expression. Always give the indexing explicitly to
  display a derived, indexed quantity:
  ```
  ampl: display {p in PROD, t in 1..T} revenue[p,t]*Sell[p,t];
  ```
  Reordering the indexing expression (`{t in 1..T, p in PROD}` vs. `{p in PROD, t in 1..T}`)
  changes the sort/grouping of rows in the output — the only way to control row ordering.
- Group several same-shape expressions into one table with a parenthesized list after one
  indexing expression: `display {p in PROD, t in 1..T} (revenue[p,t]*Sell[p,t], prodcost[p]*Make[p,t]);`
- Rows/columns are sorted alphabetically/numerically by default; declare a set `ordered`
  (`03-sets-and-indexing.md`) if you want display output to preserve your data's original order.
- Sets displayed by a *condition on variables* (e.g. `display {(i,j) in LINKS: Trans[i,j] > 500};`)
  are legal in commands even though such a condition could never appear inside a model
  declaration.

## Formatting options

| Option | Default | Effect |
|---|---|---|
| `display_1col` | 20 | max element count for one-column list format; above it, switches to compact list/table |
| `display_transpose` | 0 | bias toward/away from transposing a 2-D table (positive = more transposition) |
| `display_width` | 79 | max output line width; wide tables are cut into column-blocks |
| `gutter_width` | 3 | spacing between table columns |
| `omit_zero_rows` / `omit_zero_cols` | 0 | suppress all-zero rows/columns — invaluable for sparse assignment/flow solutions |

```
ampl: option display_1col 0;              # force compact table/list format
ampl: option omit_zero_rows 1;            # e.g. show only the nonzero assignment pairs
```

## Numeric precision: cosmetic vs. solution-altering

Two independent families — know which one you're changing:

- **`display_precision`, `display_round`, `display_eps`, `print_precision`,
  `print_round`** — affect only how numbers are *printed*; the underlying stored value is
  unchanged. `display_round n` (n ≥ 0: digits after decimal; n < 0: rounds to 10^|n|) overrides
  `display_precision` (significant digits, default 6, 0 = full precision). `display_eps`
  treats any value smaller in magnitude than the threshold as exactly zero *for display only*.
- **`solution_precision`, `solution_round`** — actually **round the values returned by the
  solver**, permanently, the moment they're received. Use these (not the display-only options)
  when tiny solver noise (`2.05994e-17` instead of exactly `0`) would otherwise corrupt a
  downstream comparison like `> 0` in a condition or a hand-summed total. This is a real,
  recurring gotcha: comparisons against solver output can silently misbehave near zero unless
  you either compare against a tolerance or set `solution_round` before `solve`.
  ```
  ampl: option solution_round 10;   # set BEFORE solve to affect the values actually stored
  ampl: solve;
  ```

## `print` and `printf`: unformatted / custom-formatted output

```
ampl: print sum {p in PROD, t in 1..T} revenue[p,t]*Sell[p,t];      # single line, space-separated
ampl: print {t in 1..T}: {p in PROD} Make[p,t];                      # one line per index member
ampl: printf "Total revenue is $%6.2f.\n", sum {p in PROD,t in 1..T} revenue[p,t]*Sell[p,t];
ampl: printf {t in 1..T}: "%3i%12.2f\n", t, sum {p in PROD} Sell[p,t];
```

- `print` items are individual values only (unlike `display`, a bare set name is not allowed —
  print its members via an indexing expression instead); nesting an indexing expression inside
  a print item *is* allowed, unlike in `display`.
  `print_separator` controls the field separator (set to a tab for spreadsheet-friendly output).
- `printf` uses C-`printf`-style format strings (`%6.2f`, `%3i`, `\n` for line breaks — output
  is not auto-broken into lines otherwise). Best for short, fixed-shape summary reports where
  the number of fields per line is known in advance; not well suited to tables with a
  variable-length row (use `display` or a script loop with `print` for that).
- Both support the same `>filename` / `>>filename` redirection as `display` (see below).

## Suffixes: bounds, slack, duals, reduced costs

A qualified name (`Var.suffix`, `Con.suffix`) works anywhere an unqualified name does. Core
suffixes to know:

| Suffix (on) | Meaning |
|---|---|
| `Var.lb`, `.ub` | bounds actually sent to the solver (post-presolve — may be tighter than the model's literal bounds) |
| `Var.slack` | distance from the variable's value to its nearer bound |
| `Var.rc` | reduced cost — marginal value with respect to the variable's own bound |
| `Con.lb`, `.body`, `.ub` | every constraint normalizes to `lb <= body <= ub`; `.body` is the current value of the variable-containing part |
| `Con.slack` | distance from `.body` to its nearer active bound; `0` means the constraint is actually binding |
| `Con` (bare name) | the constraint's dual value / marginal / shadow price |
| `ObjName` (bare name) | the objective's current value, computable even for objectives not currently selected |

```
ampl: display Buy.lb, Buy, Buy.ub, Buy.rc;
ampl: display Diet_Min.lb, Diet_Min.body, Diet_Min.ub, Diet_Min.slack;
ampl: display Time, Time.slack;     # dual value alongside slack -- classic "what's binding" check
```

**Reading duals/reduced costs**: a dual value or reduced cost is the local slope of
optimal-objective-vs-bound (piecewise linear, convex for minimize / concave for maximize) —
"cost changes by $X per unit change in this bound, near the current value." It is zero
wherever the corresponding slack is positive (a non-binding bound has no marginal effect). At a
breakpoint the true left/right slopes differ; the solver reports only one value from that
range, so treat duals/reduced costs as a local, first-order estimate — for real sensitivity
exploration, resolve with the bound changed (`let`, `08-commands-modifying-models-and-data.md`)
rather than trusting a single dual value far from the current point.

## Model-introspection commands

```
ampl: show;                    # list all component names by type
ampl: show vars;               # restrict to one type
ampl: show Total_Cost;         # print one component's declaration as AMPL currently understands it
ampl: xref demand, Trans;      # what depends on this component (directly or indirectly)?
ampl: expand Supply;           # print the fully-instantiated constraints AMPL generated
ampl: expand Trans;            # for a variable: every nonlinear/linear coefficient it appears with
ampl: solexpand Supply;        # same as expand, but the solver's post-presolve view
```

- `expand`/`show` reflect the **modeler's view** (pre-presolve); `solexpand` and the `_s`-prefixed
  generic-name family reflect the **solver's view** (post-presolve) — reach for `solexpand` when
  a solve behaves differently than `expand` output would suggest.
- **Generic synonyms** let you iterate over every variable/constraint/objective regardless of
  which declaration it came from — the standard idiom for "list every variable that's away from
  its bound" or similar cross-cutting reports:
  ```ampl
  display {j in 1.._nvars} (_varname[j], _var[j], _var[j].ub, _var[j].rc);
  display {j in 1.._nvars: _var[j] < _var[j].ub - 1e-5} _varname[j];
  ```
  Available families: `_nvars/_varname/_var`, `_ncons/_conname/_con`, `_nobjs/_objname/_obj`
  (and `_s`-prefixed post-presolve equivalents).
- `option show_stats 1;` before `solve` prints a size/sparsity summary (variables, constraints,
  nonzeros, what presolve eliminated) — a fast first check when a solve is slower or bigger than
  expected. `option times 1;` / `gentimes 1;` add AMPL-side timing detail; `_solve_time` /
  `_ampl_time` are always-available timing parameters.

## Redirecting and logging output

```
ampl: display ORIG >multi.out;      # create/overwrite; subsequent >multi.out commands append
ampl: display supply >multi.out;
ampl: close multi.out;              # stop appending
ampl: display foo >>existing.out;   # explicitly append instead of overwrite
ampl: option log_file 'session.tmp';   # mirror the whole interactive session (commands + output) to a file
```

`log_model` / `log_data` (default off) additionally echo the contents of files read via
`model`/`data`/`include` into the log. `eexit n` controls how many error messages abort a
statement (negative) or the whole session (positive) before giving up; `presolve_warnings` and
`bad_subscripts` cap how many infeasibility/bad-index messages print before truncating.

## See also

- Solver interaction internals (presolve, solve results/statuses, suffixes beyond the basics):
  `11-solver-interaction-and-suffixes.md`
- Changing data/model state that display then reports on: `08-commands-modifying-models-and-data.md`
- Automating repeated display/report sequences: `10-scripting-and-control-flow.md`
