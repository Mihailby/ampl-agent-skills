# amplpy Core Workflow: Model, Data, Solve, Results

Derived from: amplpy-best-practices.md

Source: `amplpy-best-practices.md` (Sections 2-9).

## Table of contents

- Loading a model: file vs. inline string
- Loading data: Python-native structures, DataFrames, external sources, `.dat` files
- Modifying data on an already-loaded model
- Setting AMPL and solver options
- Solving and checking `solve_result`
- Retrieving and exporting results
- Fetching arbitrary values via AMPL expressions
- Batch-solving multiple models/solvers
- A sensitivity-analysis pattern

## Loading a model

**Prefer a `.mod` file, read with `ampl.read()`**, over inlining the model as a Python
string — this is the recommended default for anything beyond a trivial model, for the
usual reasons a separate file beats an embedded string (readability, diffability, reuse
outside Python):

```python
ampl.read('model.mod')
```

Inline loading (`ampl.eval(r"""...""")`, or `%%ampl_eval` cell magic in a notebook) is
explicitly called out as acceptable **only** for small models (rule of thumb: under ~20
lines) — for anything larger it's flagged as bad practice for readability, even though
AMPL will happily accept it. In Colab/Jupyter, `%%writefile model.mod` followed by
`ampl.read('model.mod')` gets you the "real file" workflow without leaving the notebook.

## Loading data

`amplpy` accepts data from essentially any Python-native structure, which is the main
argument for preferring Python-side data loading over AMPL `.dat` files in most cases
(see the closing recommendation below).

### Lists → sets

```python
ampl.set["FOOD"] = ['BEEF', 'CHK', 'FISH', ...]     # direct assignment
ampl.set["FOOD"] = list(foods.keys())                # from a dict's keys
A = ampl.get_set('A'); A.set_values([1, 2, 3])       # via the Set object
```

### Dictionaries → parameters (and sets, via `.keys()`)

```python
cost = {"BEEF": 3.59, "CHK": 2.59, ...}
ampl.set["FOOD"] = cost.keys()
ampl.param["cost"] = cost
```

For composite/tuple-indexed data (a 2-D parameter like nutrient amounts per food),
dictionary comprehensions extracting fields from richer source dictionaries, and a
dictionary keyed by tuples for a `LINKS`-style set/param pair, are the standard pattern:

```python
amt = {("A", "BEEF"): 60, ("C", "BEEF"): 20, ...}    # keys are (nutrient, food) tuples
ampl.set['LINKS'] = set(amt.keys())
ampl.param['amt'] = amt
```

### pandas / polars DataFrames → sets + parameters together

`ampl.set_data(df, "SET_NAME")` is the single most convenient loading path when data is
already tabular: it defines the named indexing set **and** every parameter whose columns
appear in the DataFrame, in one call, provided the DataFrame's index matches what should
become the AMPL set:

```python
food_df = pd.DataFrame([...], columns=["FOOD","cost","f_min","f_max"]).set_index("FOOD")
ampl.set_data(food_df, "FOOD")     # defines set FOOD, and params cost/f_min/f_max over FOOD
```

For a 2-D parameter matrix (rows/columns both meaningful), build the `LINKS`-style tuple
set from the DataFrame's own index/columns, filtering out zero/absent entries, then
extract the corresponding value dictionary before assigning:

```python
ampl.set['LINKS'] = [(n, f) for n in amt_df.index for f in amt_df.columns if amt_df.loc[n,f] != 0]
filtered = {(n, f): amt_df.loc[n, f] for n, f in ampl.set['LINKS']}
ampl.get_parameter("amt").set_values(filtered)
```

**polars** DataFrames aren't accepted directly — `amplpy` expects pandas — convert first:
`pl_df.to_pandas().set_index(...)` before calling `set_data`.

### External sources, all funneled through pandas

Because pandas can read almost anything and `amplpy` accepts pandas DataFrames directly,
external sources become a two-step "read into pandas, then `set_data`" pattern regardless
of origin: JSON (`json.load` → `pd.DataFrame`), CSV (`pd.read_csv`), Excel with multiple
sheets (`pd.read_excel(path, sheet_name=None)` returns a dict of DataFrames, one per
sheet), Google Sheets (fetch the sheet's CSV-export URL with `pd.read_csv`), PostgreSQL
(`sqlalchemy` + `pd.read_sql`). This uniformity is the practical payoff of routing
everything through DataFrames rather than writing a bespoke loader per source type.

**Large datasets (over ~10 million rows)**: prefer AMPL's native table handlers
(`amplcsv`, `amplxl`, `eodbc` — the `table`/`read table` mechanism in
`../core-language/07-database-access-and-tables.md`) to load data *directly* into AMPL,
bypassing the Python/pandas round-trip for performance; `amplpy` remains useful
alongside this for updating data incrementally and retrieving solutions.

### `.dat` files

`ampl.read_data('data.dat')` still works and is sometimes appropriate, but the guidance
here is to prefer Python-native/DataFrame loading (above) for better maintainability in an
amplpy-driven workflow — reserve `.dat` files for pure-AMPL scripts/workflows or migration
scenarios, not as the default in new Python-driven projects.

## Modifying data on an already-loaded model

Two distinct operations, worth keeping straight:

```python
cost.set_values([3, 5, 5, 6, 1, 2, 5.01, 4.55])         # REPLACE ALL entries, in AMPL's index order
ampl.param['cost'] = {"BEEF": 5.01, "HAM": 4.55}        # UPDATE SPECIFIC entries, others untouched
# equivalently:
ampl.get_parameter("cost").set_values({"BEEF": 5.01, "HAM": 4.55})
```

A list-based `set_values()` call replaces every entry and relies on matching AMPL's
declared index order exactly (fragile if the index order isn't obvious) — a
dictionary-based assignment is index-explicit and safer for partial updates, and is
generally the better default unless you really do intend a full, order-matched
replacement. The same `set_values()` pattern applies to sets
(`ampl.set['TIME'] = list(range(10))`).

## Setting options

```python
ampl.option['solver'] = 'highs'
ampl.option['mp_options'] = 'outlev=0 lim:time=20'      # options shared by MP-framework solvers
ampl.option['presolve'] = 90                             # presolve iteration cap (default 10)
# equivalent explicit-method form:
ampl.set_option('solver', 'highs')
```

`mp_options` configures options common to the newer "MP"-framework solver drivers;
solver-specific option strings (`gurobi_options`, `highs_options`, etc.) are also
accepted and take precedence for that solver. Full option list:
https://dev.ampl.com/ampl/options.html; full solver-option guide:
https://dev.ampl.com/solvers/index.html.

## Solving

```python
ampl.solve(solver='highs')                    # explicit solver argument
# or:
ampl.option["solver"] = "highs"; ampl.solve(verbose=False)

assert ampl.solve_result == "solved"          # fail fast if the solve didn't succeed
```

`solve_result`/`solve_result_num` use the same code ranges as the underlying AMPL
parameters (`../core-language/11-solver-interaction-and-suffixes.md`): 0-99 `solved`,
100-199 `solved?` (result likely but with a caveat), 200-299 `infeasible`, 300-399
`unbounded`, 400-499 `limit` (stopped by a limit you set), 500-599 `failure`. Checking
this — not just assuming success because no exception was raised — is standard practice
in any amplpy script meant to run unattended.

**Debugging infeasibility**: some solvers (Gurobi shown) can report an Irreducible
Infeasible Subsystem directly:

```python
ampl.solve(solver="gurobi", gurobi_options="outlev=1 iis=1")
if ampl.solve_result == "infeasible":
    var_iis, con_iis = ampl.get_iis()
```

## Retrieving and exporting results

```python
solution = ampl.get_solution(flat=False, zeros=False)   # nested dict; zeros=False omits 0-valued entries
```

From there, conversion to any downstream format is a one-liner: `pd.DataFrame.from_dict(...)`
for pandas, `pl.from_dict(...)` for polars, `json.dump(...)` for JSON, `.to_csv(...)` /
`ExcelWriter` for CSV/Excel, `gspread`+`set_with_dataframe` for Google Sheets,
`DataFrame.to_sql(...)` for PostgreSQL. The common thread: `get_solution()` is the single
retrieval point, everything after it is ordinary pandas/Python I/O — no amplpy-specific
export API to learn per destination format.

## Fetching arbitrary values via AMPL expressions

For values that aren't a single variable/parameter dump but a computed expression, use
`ampl.get_data()` with an AMPL indexing expression as a string, then convert:

```python
df = ampl.get_data("{j in FOOD} 100*Buy[j]/Buy[j].ub").to_pandas()
```

This is the most flexible read path in amplpy: you write ordinary AMPL syntax (any valid
indexed expression, including suffix references like `.ub`) as a Python string, and get
back a DataFrame — useful whenever what you want isn't a raw entity's value but a derived
quantity, without needing to declare a new AMPL variable/parameter just to expose it.
Equivalently, an individual entity's data is available directly: `ampl.var['Buy'].to_pandas()`
/ `.to_dict()`.

## Batch-solving multiple models/solvers

The Python-side equivalent of the AMPL-native "load model once, loop `data` + `reset
data` per instance" pattern (`../tutorials-and-examples/worked-example-tutorial.md`,
Stages 3 and 5): wrap the repeated "set solver, solve" step in a small helper so a driver
script can loop cleanly over combinations of models and solvers, calling `ampl.reset()`
between genuinely different models (not `ampl.eval("reset data;")`, which is for reusing
the same model with new data — see that file for the distinction):

```python
def set_n_solve(ampl, solver_name):
    ampl.set_option("solver", solver_name)
    ampl.solve()

for model_file in ["lemonade.mod", "mulled_wine.mod"]:
    ampl.read(model_file)
    for solver_name in ["cbc", "highs"]:
        set_n_solve(ampl, solver_name)
        # ... retrieve/print results (get_solution(), get_data(), etc.) ...
    ampl.reset()          # different model coming next in the outer loop
```

Keeping `set_n_solve` (or any solve-driving helper) as its own function, rather than
inlining `ampl.solve()` at every call site, pays off the same way `<solver>_solve.run`
does in pure AMPL: swapping a solver, adding logging, or adding a timeout becomes a
one-place edit instead of a find-and-replace across the script.

## Sensitivity analysis pattern

A simple, general-purpose parameter sweep: perturb one parameter by a set of percentage
deltas, re-solve, and collect the objective at each delta.

```python
def sensitivity_analysis_parameter(ampl, param_name, deltas):
    results = {}
    original = ampl.param[param_name].value()
    for delta in deltas:
        ampl.param[param_name] = original * (1 + delta)
        ampl.solve()
        results[delta] = ampl.obj['Profit'].value()
    return results
```

Straightforward to adapt to any scalar parameter and any objective name; for multiple
parameters swept jointly, or for re-solving many independent scenarios efficiently,
consider whether an AMPL-side `for`/`repeat` script
(`../core-language/10-scripting-and-control-flow.md`) would be more efficient than
looping from the Python side, since each `ampl.solve()` call round-trips through the AMPL
interpreter.

## See also

- Installing amplpy and its solver modules: `getting-started-and-installation.md`
- Full method-by-method reference for `AMPL`, `Entity`, `Variable`, `Constraint`,
  `Objective`, `Set`, `Parameter`: `api-reference-classes.md`
- AMPL's native `table`/`read table` mechanism, the non-Python equivalent of `set_data`:
  `../core-language/07-database-access-and-tables.md`
- Data-validation habit (`check` statements) from the pure-AMPL side, worth mirroring at
  the pandas/pre-load stage: `../core-language/02-style-and-best-practices.md`
