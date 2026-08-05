# amplpy API Patterns Reference

## Installation and Setup

```bash
# Install Python API for AMPL
python -m pip install amplpy --upgrade

# Install solver modules (install whichever solvers you need)
python -m amplpy.modules install highs gurobi xpress cplex

# Activate your license (free licenses at ampl.com/ce or ampl.com/courses)
python -m amplpy.modules activate <your-license-uuid>
```

```python
from amplpy import AMPL
import pandas as pd

ampl = AMPL()
```

If not using `amplpy.modules` and AMPL is not in the system path:
```python
from amplpy import AMPL, add_to_path
add_to_path(r"full path to the AMPL installation directory")
ampl = AMPL()
```

## Loading Models

**From a file (preferred):**
```python
ampl.read("model.mod")
```

**Inline (acceptable for quick prototyping):**
```python
ampl.eval(r"""
    set CITIES;
    param distance {CITIES, CITIES} >= 0;
    var X {CITIES} binary;
    minimize Total: sum {i in CITIES, j in CITIES} distance[i,j] * X[i] * X[j];
""")
```

`ampl.read()` is the standard approach. Use `ampl.eval()` only for quick inline definitions
or when files are inconvenient (e.g., notebook environments).

## Data Loading

### From pandas DataFrames

**Pattern 1: `ampl.set_data(df, "SET_NAME")`**

The DataFrame index becomes the set members. Columns become parameter values for parameters
indexed over that set.

```python
food_df = pd.DataFrame([
    ("BEEF", 3.59, 2, 10),
    ("CHK",  2.59, 2, 10),
], columns=["FOOD", "cost", "f_min", "f_max"]).set_index("FOOD")

ampl.set_data(food_df, "FOOD")
# This initializes set FOOD = {BEEF, CHK} and sets param cost, f_min, f_max values.
```

**Pattern 2: `ampl.get_parameter("p").set_values(df)`**

For parameters indexed over multiple sets (e.g., matrices), use a DataFrame where the index
and columns correspond to the set indices.

```python
import numpy as np

amt_df = pd.DataFrame(
    np.array([[60, 8], [20, 0]]),
    columns=["BEEF", "CHK"],       # FOOD index
    index=["A", "C"],              # NUTR index
)
ampl.get_parameter("amt").set_values(amt_df)
```

### From Python dicts and lists

```python
# Sets
ampl.set["FOOD"] = ["BEEF", "CHK", "FISH"]

# Scalar parameters indexed over one set
ampl.param["cost"] = {"BEEF": 3.59, "CHK": 2.59, "FISH": 2.29}

# Parameters indexed over multiple sets
ampl.param["amt"] = {
    ("A", "BEEF"): 60,
    ("A", "CHK"):  8,
    ("C", "BEEF"): 20,
    ("C", "CHK"):  0,
}

# Updating specific parameter values
cost = ampl.get_parameter("cost")
cost.set_values({"BEEF": 5.01, "HAM": 4.55})
```

### Performance Considerations

amplpy handles DataFrames with millions of rows efficiently. For datasets exceeding ~10 million
rows, consider AMPL table handlers that load data directly without passing through Python:

- `amplcsv` -- CSV files
- `amplxl` -- Excel XLSX files
- `eodbc` -- Databases (MySQL, PostgreSQL, etc.) via ODBC

You can still use the amplpy API to update data and retrieve solutions while using table handlers
for the initial bulk data load.

## Solving

```python
# Set solver
ampl.option["solver"] = "highs"

# Set solver-specific options
ampl.option["highs_options"] = "outlev=1 timelimit=300"

# General pattern for solver options:
# ampl.option[f"{solver}_options"] = "option_string"

# Solve
ampl.solve()

# Always check solve status
assert ampl.solve_result == "solved"
# Other possible solve_result values: "infeasible", "unbounded", "limit", etc.
```

### Solver option examples

```python
# HiGHS
ampl.option["highs_options"] = "outlev=1 timelimit=300"

# Gurobi
ampl.option["gurobi_options"] = "outlev=1 timelimit=300 mipgap=0.01 threads=8"

# CPLEX
ampl.option["cplex_options"] = "outlev=1 timelimit=300 mipgap=0.01"

# COPT
ampl.option["copt_options"] = "outlev=1 timelimit=300"
```

## Retrieving Results

### Objective value
```python
total_cost = ampl.get_objective("Total_Cost")
print(total_cost.value())
```

### Variable values as pandas DataFrame
```python
buy = ampl.get_variable("Buy")
df = buy.get_values().to_pandas()
print(df)
```

### Constraint information
```python
diet = ampl.get_constraint("Diet")
df = diet.get_values().to_pandas()
```

### Arbitrary AMPL expressions
```python
# Evaluate any AMPL expression and get results as a DataFrame
df = ampl.get_data("{j in FOOD} 100*Buy[j]/Buy[j].ub").to_pandas()
```

## Modifying Data and Re-solving

```python
# Update specific parameter values
ampl.get_parameter("cost").set_values({"BEEF": 5.01, "HAM": 4.55})

# Or using the shorthand
ampl.param["cost"] = {"BEEF": 5.01, "HAM": 4.55}

# Re-solve
ampl.solve()
assert ampl.solve_result == "solved"
print(f"New objective: {ampl.get_objective('Total_Cost').value()}")
```

## Common Patterns

### Sensitivity analysis loop
```python
results = {}
for budget in [100, 200, 300, 400, 500]:
    ampl.param["max_budget"] = budget
    ampl.solve()
    if ampl.solve_result == "solved":
        results[budget] = ampl.get_objective("Profit").value()

results_df = pd.DataFrame(list(results.items()), columns=["Budget", "Profit"])
```

### Multiple scenarios
```python
for scenario_name, scenario_data in scenarios.items():
    ampl.get_parameter("demand").set_values(scenario_data)
    ampl.solve()
    if ampl.solve_result == "solved":
        print(f"{scenario_name}: {ampl.get_objective('Cost').value():.2f}")
```
