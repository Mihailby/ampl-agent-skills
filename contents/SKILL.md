---
name: ampl-optimization
description: >
  Write AMPL mathematical optimization models and accompanying Python (amplpy) code following
  modern best practices from AMPL Optimization Inc. Use this skill whenever the user asks for help
  with AMPL, mathematical programming models in AMPL, amplpy, the AMPL Python API, linear programming,
  mixed-integer programming, optimization modeling, or any task involving formulating and solving
  optimization problems with AMPL. Also trigger when the user mentions solvers commonly used with
  AMPL (HiGHS, Gurobi, CPLEX, COPT, Xpress, Mosek, SCIP, CBC, BARON) in an AMPL context, or when
  they reference .mod files, AMPL syntax, or want to convert a mathematical formulation into code.
  Even if the user just says "write an optimization model" or "solve this LP/MIP/QP," consider
  triggering this skill if AMPL is a reasonable choice.
---

# AMPL Optimization Skill

This skill teaches Claude how to write modern AMPL optimization code the way AMPL Optimization
intends it to be written. Claude's training data contains outdated AMPL patterns. This skill
corrects those patterns and introduces the modern workflow.

## The Modern AMPL Workflow

The modern AMPL workflow has exactly two components:

1. **A `.mod` file** containing the model definition: sets, parameters, variables, objective(s),
   and constraints. Data is *not* embedded in the `.mod` file.
2. **A Python file** using `amplpy` that handles everything else: loading the model, loading data
   (typically from pandas DataFrames or Python dicts), configuring the solver, solving, checking
   results, and any application logic.

There are **no `.dat` files** and **no `.run` files**. Those are legacy patterns from when AMPL was
used as a standalone scripting environment. Today, Python is the orchestration layer.

### What Claude must NOT do

- **Never generate `.dat` files.** Data goes through amplpy from Python data structures.
- **Never generate `.run` files.** Solve commands and scripting logic belong in Python via amplpy.
- **Never recommend the AMPL IDE.** The recommended editor is the official AMPL VS Code extension.
- **Never use old-style linearizations** (big-M, manual indicator reformulations) when AMPL MP
  can handle the expression directly. MP-based solvers understand logical, conditional, counting,
  and piecewise-linear expressions natively or via automatic reformulation. See the
  [MP expressions reference](#ampl-mp-expressions) and `references/mp-expressions.md`.
- **Never use `ampl.eval()` for data loading.** Use `ampl.set_data()`, `ampl.param[...]`, or
  `ampl.set[...]` from Python.

### What Claude SHOULD do

- Write clean `.mod` files with the model definition only (sets, params, vars, objectives,
  constraints).
- Write a Python file using `amplpy` to load data, solve, and retrieve results.
- Use AMPL MP expressions (logical operators, conditionals, counting, piecewise-linear, etc.)
  instead of manually linearizing constraints. This is one of AMPL's biggest modern advantages --
  let the MP layer handle reformulation.
- Default to HiGHS as the solver (open-source, handles LP/MIP/QP), but mention that commercial
  solvers like Gurobi, CPLEX, COPT, Xpress, and Mosek are available for larger or specialized
  problems.
- Use `ampl.read("model.mod")` to load the model (preferred over `ampl.eval()`).
  `ampl.eval()` is acceptable for quick inline definitions but `ampl.read()` is the standard.
- Always check `ampl.solve_result == "solved"` after solving.

### Common AMPL Syntax Pitfalls

These are mistakes Claude is prone to making. Watch out for them:

- **`ord()` requires an `ordered` or `circular` set.** If you need `ord(i)` to iterate over
  unique pairs (e.g., `ord(j1) < ord(j2)`), declare the set as `set JOBS ordered;`. Using
  `ord()` on a plain `set` is an error.
- **Double-inequality constraints** like `a <= expr <= b` are valid AMPL syntax and preferred
  over writing two separate constraints. Use them.
- **Semicolons terminate declarations**, not individual lines. A multi-line constraint is a single
  statement ending with one semicolon.
- **`subject to` and `subj to` and `s.t.`** are all valid abbreviations for constraints.

## Output Format

When generating AMPL code, produce **two separate code blocks or artifacts**:

1. **The `.mod` file** -- model definition only.
2. **The Python file** -- amplpy orchestration code.

Always label them clearly so the user knows which is which.

## amplpy Patterns

Read `references/amplpy-patterns.md` for detailed API examples. The key patterns are summarized
here.

### Installation

```bash
# Install Python API for AMPL
python -m pip install amplpy --upgrade

# Install solver modules (install whichever solvers you need)
python -m amplpy.modules install highs gurobi xpress cplex

# Activate your license (free licenses at ampl.com/ce or ampl.com/courses)
python -m amplpy.modules activate <your-license-uuid>
```

### Basic Structure

```python
from amplpy import AMPL
import pandas as pd

ampl = AMPL()

# Load model
ampl.read("model.mod")

# Load data from pandas DataFrames
ampl.set_data(df, "SET_NAME")                    # loads a DF and initializes a set
ampl.get_parameter("param_name").set_values(df)   # loads values into an existing param

# Alternative: load data from dicts/lists
ampl.set["SET_NAME"] = ["a", "b", "c"]
ampl.param["param_name"] = {"a": 1, "b": 2, "c": 3}

# Configure solver
ampl.option["solver"] = "highs"
# Optional: pass solver-specific options
ampl.option["highs_options"] = "outlev=1"

# Solve
ampl.solve()
assert ampl.solve_result == "solved"

# Retrieve results
obj_value = ampl.get_objective("ObjName").value()
results_df = ampl.get_variable("VarName").get_values().to_pandas()

# Retrieve arbitrary expressions
expr_df = ampl.get_data("{i in SET} expr[i]").to_pandas()
```

### Data Loading Patterns

**From pandas DataFrames (recommended for structured data):**
- `ampl.set_data(df, "SET_NAME")` -- the DataFrame index becomes the set members, columns
  become parameter values.
- `ampl.get_parameter("p").set_values(df)` -- for parameters indexed over multiple sets
  (e.g., a matrix), use a DataFrame where rows/columns correspond to set indices.

**From Python dicts (convenient for small data or programmatic construction):**
- `ampl.set["S"] = [...]` -- initialize a set.
- `ampl.param["p"] = {...}` -- set parameter values with a dict mapping indices to values.
- For multi-dimensional parameters: `ampl.param["p"] = {("i1","j1"): val, ...}`

**Performance note:** amplpy handles DataFrames with millions of rows efficiently. For datasets
exceeding ~10 million rows, consider AMPL's table handlers (`amplcsv`, `amplxl`, `eodbc`) which
load data directly into AMPL from CSV, Excel, or databases without passing through Python.

### Result Retrieval

```python
# Objective value
ampl.get_objective("Cost").value()

# Variable values as pandas DataFrame
ampl.get_variable("X").get_values().to_pandas()

# Constraint duals
ampl.get_constraint("Balance").get_values().to_pandas()

# Arbitrary AMPL expressions
ampl.get_data("{i in NODES} Capacity[i] - Flow[i]").to_pandas()
```

### Solver Configuration

```python
ampl.option["solver"] = "highs"

# Solver-specific options use the pattern: {solver}_options
ampl.option["highs_options"] = "outlev=1 timelimit=300"
ampl.option["gurobi_options"] = "outlev=1 timelimit=300 mipgap=0.01"
```

## AMPL MP Expressions

This is critical. AMPL's MP solver interface library allows a much richer set of expressions
than what many users (and Claude's training data) know about. MP-based solvers can handle these
expressions natively or via automatic reformulation -- there is no need to manually linearize them.

**All current AMPL solvers are MP-based:** Gurobi, CPLEX, COPT, Xpress, Mosek, BARON, HiGHS,
cuOpt, CBC, SCIP, GCG.

Read `references/mp-expressions.md` for the full catalog of supported expressions with examples.
The major categories are:

### Conditional Operators

Use `if-then-else` with variables in the condition, and implication operators (`==>`, `<==`,
`<==>`). No need for big-M reformulations.

```ampl
# Variable-dependent conditional in objective
minimize TotalCost:
   sum {j in JOBS, k in MACHINES}
      if MachineForJob[j] = k then cost[j,k];

# Implication constraints -- MP handles these directly
subject to Least_Use {j in SCHEDS}:
   Use[j] = 1 ==> Work[j] >= least_assign else Work[j] = 0;
```

### Logical Operators

Use `or`, `and`, `not`, `exists`, `forall` directly in constraints.

```ampl
# Disjunctive constraint -- no big-M needed
subject to No_Conflict {i1 in JOBS, i2 in JOBS: ord(i1) < ord(i2)}:
   Start[i2] >= Start[i1] + t_offset[i1,i2] or
   Start[i1] >= Start[i2] + t_offset[i2,i1];

# Iterated logical operators
subject to Multi {i in ORIG, j in DEST}:
   forall {p in PROD} Trans[i,j,p] = 0 or
   minload <= sum {p in PROD} Trans[i,j,p] <= limit[i,j];
```

### Counting Operators

`count`, `atleast`, `atmost`, `exactly`, `numberof`, `alldiff` -- express cardinality and
distinctness constraints directly.

```ampl
subject to Min_Serve {i in ORIG}:
   atleast minserve {j in DEST} (Ship[i,j] >= minload);

subject to OnePersonPerPosition:
   alldiff {i in 1..nPeople} Pos[i];
```

### Piecewise-Linear Functions

`abs`, `min`, `max`, and the `<< breakpoints; slopes >>` operator. Set `option pl_linearize 0`
for MP solvers to let them handle piecewise-linear expressions natively.

```ampl
maximize Total_Profit:
   sum {p in PROD, t in 1..T} revenue[p,t]*Sell[p,t] -
   sum {t in 1..T} time_penalty[t] * abs(Use[t] - avail_min[t]);

minimize Max_Cost:
   max {i in PEOPLE} sum {j in PROJECTS} cost[i,j] * Assign[i,j];
```

### Comparison and Relational Operators

Use `!=` and `alldiff` directly. MP handles the transformation.

```ampl
subject to Different_Colors {(c1,c2) in Neighbors}:
   Color[c1] != Color[c2];
```

### Nonlinear Functions

MP-based solvers support `exp`, `log`, `sin`, `cos`, `tan`, `sinh`, `cosh`, `tanh` and their
inverses. For linear/MIP solvers, these are approximated via piecewise-linear functions
automatically. For Gurobi 12+, native nonlinear handling via spatial branching is available.

### Key MP Modeling Advice

- **Prefer MP logical forms over manual linearization.** Writing `x > 0 ==> y >= 5` is clearer,
  less error-prone, and often more efficient than introducing big-M constraints by hand.
- **Set `option pl_linearize 0;`** when using MP-based solvers with piecewise-linear expressions,
  so the solver handles them natively rather than AMPL linearizing first.
- **Bound your variables tightly** when using piecewise-linear approximations of nonlinear
  functions. Tight bounds within [-1e4, 1e4] improve numerical stability.
- **Use indicator constraints** when possible: `binary_var = 1 ==> constraint` is handled
  natively by many solvers (Gurobi, CPLEX, COPT, Xpress).

## Example: Complete Modern Workflow

Here is a complete example showing the recommended two-file approach for a diet problem.

**diet.mod:**
```ampl
set NUTR;
set FOOD;

param cost {FOOD} > 0;
param f_min {FOOD} >= 0;
param f_max {j in FOOD} >= f_min[j];

param n_min {NUTR} >= 0;
param n_max {i in NUTR} >= n_min[i];

param amt {NUTR, FOOD} >= 0;

var Buy {j in FOOD} >= f_min[j], <= f_max[j];

minimize Total_Cost: sum {j in FOOD} cost[j] * Buy[j];

subject to Diet {i in NUTR}:
   n_min[i] <= sum {j in FOOD} amt[i,j] * Buy[j] <= n_max[i];
```

**solve_diet.py:**
```python
from amplpy import AMPL
import pandas as pd

ampl = AMPL()
ampl.read("diet.mod")

# Prepare data as pandas DataFrames
food_df = pd.DataFrame([
    ("BEEF", 3.59, 2, 10),
    ("CHK",  2.59, 2, 10),
    ("FISH", 2.29, 2, 10),
    ("HAM",  2.89, 2, 10),
    ("MCH",  1.89, 2, 10),
    ("MTL",  1.99, 2, 10),
    ("SPG",  1.99, 2, 10),
    ("TUR",  2.49, 2, 10),
], columns=["FOOD", "cost", "f_min", "f_max"]).set_index("FOOD")

nutr_df = pd.DataFrame([
    ("A",   700, 20000),
    ("C",   700, 20000),
    ("B1",  700, 20000),
    ("B2",  700, 20000),
    ("NA",    0, 50000),
    ("CAL", 16000, 24000),
], columns=["NUTR", "n_min", "n_max"]).set_index("NUTR")

amt_data = {
    ("A",  "BEEF"): 60,  ("A",  "CHK"): 8,   ("A",  "FISH"): 8,   ("A",  "HAM"): 40,
    ("A",  "MCH"):  15,  ("A",  "MTL"): 70,  ("A",  "SPG"):  25,  ("A",  "TUR"): 60,
    ("C",  "BEEF"): 20,  ("C",  "CHK"): 0,   ("C",  "FISH"): 10,  ("C",  "HAM"): 40,
    ("C",  "MCH"):  35,  ("C",  "MTL"): 30,  ("C",  "SPG"):  50,  ("C",  "TUR"): 20,
    ("B1", "BEEF"): 10,  ("B1", "CHK"): 20,  ("B1", "FISH"): 15,  ("B1", "HAM"): 35,
    ("B1", "MCH"):  15,  ("B1", "MTL"): 15,  ("B1", "SPG"):  25,  ("B1", "TUR"): 15,
    ("B2", "BEEF"): 15,  ("B2", "CHK"): 20,  ("B2", "FISH"): 10,  ("B2", "HAM"): 10,
    ("B2", "MCH"):  15,  ("B2", "MTL"): 15,  ("B2", "SPG"):  15,  ("B2", "TUR"): 10,
    ("NA", "BEEF"): 928, ("NA", "CHK"): 2180,("NA", "FISH"): 945, ("NA", "HAM"): 278,
    ("NA", "MCH"):  1182,("NA", "MTL"): 896, ("NA", "SPG"):  1329,("NA", "TUR"): 1397,
    ("CAL","BEEF"): 295, ("CAL","CHK"): 770, ("CAL","FISH"): 440, ("CAL","HAM"): 430,
    ("CAL","MCH"):  315, ("CAL","MTL"): 400, ("CAL","SPG"):  379, ("CAL","TUR"): 450,
}

# Load data into AMPL
ampl.set_data(food_df, "FOOD")
ampl.set_data(nutr_df, "NUTR")
ampl.get_parameter("amt").set_values(amt_data)

# Solve
ampl.option["solver"] = "highs"
ampl.solve()
assert ampl.solve_result == "solved"

# Results
print(f"Total cost: {ampl.get_objective('Total_Cost').value():.2f}")
print(ampl.get_variable("Buy").get_values().to_pandas())
```

## Solver Selection Guidance

Default to **HiGHS** (open-source, handles LP/MIP/QP). Choose a solver based on your problem
type using the tables below.

### Linear-Quadratic Solvers (MP-based)

All of these solvers use the MP interface and support the full range of MP expressions
(logical, conditional, counting, piecewise-linear, etc.) via automatic reformulation.

| Solver   | LP | MILP | QP | MIQP | Convex (MI)QCP | Non-convex (MI)QCP | Conic | MINLP |
|----------|:--:|:----:|:--:|:----:|:--------------:|:------------------:|:-----:|:-----:|
| Gurobi   | Y  | Y    | Y  | Y    | Y              | Y                  | Y     | Y     |
| Xpress   | Y  | Y    | Y  | Y    | Y              | Y                  | Y     | Y     |
| CPLEX    | Y  | Y    | Y  | Y    | Y              | --                 | Y     | --    |
| COPT     | Y  | Y    | Y  | Y    | Y              | --                 | Y     | --    |
| Mosek    | Y  | Y    | Y  | Y    | Y              | --                 | Y*    | --    |
| SCIP     | Y  | Y    | Y  | Y    | Y              | Y                  | Y     | Y     |
| GCG      | Y  | Y    | Y  | Y    | Y              | Y                  | Y     | Y     |
| HiGHS    | Y  | Y    | Y  | --   | --             | --                 | --    | --    |
| CBC      | Y  | Y    | Y  | Y    | --             | --                 | --    | --    |
| cuOpt    | Y  | Y    | -- | --   | --             | --                 | --    | --    |

*Mosek: specialized conic support (SOCP, SDP, exponential cones).

### Nonlinear Solvers

These solvers handle general nonlinear problems. Those marked with MP2NL access MP features
through the MP-to-NL translation layer.

| Solver   | NLP | MINLP | Global | MP support     |
|----------|:---:|:-----:|:------:|----------------|
| Knitro   | Y   | Y     | --     | via MP2NL      |
| BARON    | Y   | Y     | Y      | native MP      |
| Lindo    | Y   | Y     | Y      | via MP2NL      |
| Raposa   | Y   | Y     | Y      | via MP2NL      |
| LGO      | Y   | --    | Y      | via MP2NL      |
| CONOPT   | Y   | --    | --     | via MP2NL      |
| LOQO     | Y   | --    | --     | via MP2NL      |
| MINOS    | Y   | --    | --     | via MP2NL      |
| SNOPT    | Y   | --    | --     | via MP2NL      |
| Ipopt    | Y   | --    | --     | via MP2NL      |
| Bonmin   | Y   | Y     | --     | via MP2NL      |
| Couenne  | Y   | Y     | Y      | via MP2NL      |

### Quick Solver Selection Guide

- **LP/MIP (default):** HiGHS (open-source), or Gurobi/CPLEX/COPT/Xpress (commercial, faster
  on large instances)
- **QP/MIQP:** HiGHS (LP+QP only), CBC, or any commercial solver
- **Convex QCP / SOCP / Conic:** Mosek (best for conic), Gurobi, CPLEX, COPT, Xpress
- **Non-convex QCP:** Gurobi, Xpress, SCIP, GCG
- **MINLP:** Gurobi (v12+), Xpress, SCIP, GCG, BARON, Bonmin, Couenne
- **General NLP:** Knitro, Ipopt (open-source), CONOPT, SNOPT, MINOS
- **Global optimization:** BARON, Lindo, Raposa, Couenne, LGO
- **GPU-accelerated LP/MIP:** cuOpt (NVIDIA)
