# Style and Naming Conventions

Derived from: ampl-style-guide.md

Source: `ampl-style-guide.md` (AMPL Optimization's published style guide).

This is house style from AMPL's own docs, not a language requirement — AMPL enforces
none of this. Worth following anyway for any model beyond a one-off script, since
scalability/collaboration is exactly where inconsistent naming and flat, unsectioned
files become painful.

## Naming conventions by entity type

| Entity | Case | Example |
|---|---|---|
| Sets | `CONSTANT_CASE`, plural nouns | `WAREHOUSES`, `COST_TYPES` |
| Parameters | `snake_case` | `cost_per_unit`, `demand_per_product` |
| Variables | `PascalCase`, name the decision | `Shipment`, `InventoryLevel` |
| Constraints | `PascalCase`, name the restriction | `CapacityLimit`, `AlreadyFinished` |
| Objectives | `PascalCase`, action-oriented | `TotalProfit`, `TotalCost` |

The distinct casing per entity type is the load-bearing convention here: it lets a
reader tell a set from a variable from a constraint at a glance without cross-checking
declarations, which matters once a model has more than a handful of components.

General principles: prefer descriptive names (`warehouse_capacity`, not `cap`) over
short ones — short names (`i`, `j`, `p`) are fine for dummy indices inside a `sum`/`for`
where brevity aids reading a dense expression, but not for the entities themselves.
Align names to domain vocabulary (`shipment_volume`, not `volume`). Never reuse a name
across entity types (don't have both a `cost` parameter and a `Cost` variable in the
same model — differs only in case, which is exactly the kind of near-collision the
casing convention exists to prevent). Use `_` to join words in `snake_case`/`CONSTANT_CASE`
names; rely on the case change itself (not underscores) to join words in `PascalCase`.

## Model structure

Organize a `.mod` file in the conventional declaration order — sets, then parameters,
then variables, then constraints, then objective(s) — even though AMPL itself allows
any order (each name just needs to be declared before first use). A reader expects to
find, e.g., all parameters together rather than interleaved with constraints.

Use blank lines, indentation, and line breaks freely; AMPL's parser is whitespace-
insensitive, so formatting is pure communication with the next reader. For a long
`sum`-based objective or constraint body, break after the indexing expression rather
than mid-expression:

```ampl
maximize TotalProfit:
   sum {p in PRODUCTS, w in WAREHOUSES}
      profit[p, w] * Shipment[p, w];
```

Align repeated, similarly-shaped declarations in a block (matching comment columns,
matching brace positions) so the *differences* between adjacent lines are what stands
out:

```ampl
param demand {1..n_step} >= 0;   # Demand values at each price step
param price  {1..n_step} >= 0;   # Price values for each demand step
```

## Comments

Three granularities, used for different purposes:

- **Inline** (`# short note`) — sparingly, only where a line's intent isn't obvious
  from its name alone.
- **Section/assumption comments** — a line or two above a group of related
  declarations, documenting a non-obvious constraint on the data (e.g. "Assumption:
  unit_cost is always non-negative").
- **Header block** — at the top of a model file that will be revised over time:
  name, purpose, version, last-updated date. Cheap insurance against "which version of
  this model am I looking at" once a model has been through several revisions.

## Validate data early with `check`

Add `check` statements for feasibility assumptions the model depends on but can't
enforce structurally — catching a violated assumption at `data`/`check` time, with a
clear message, beats discovering it as a mysterious infeasible or wrong-looking
solution later:

```ampl
check: sum {p in PRODUCTS} demand[p] <= total_capacity;
```

(Full `check` statement syntax: `12-syntax-quick-reference.md`, section A.7.1.) The
style guide also notes this same validation can be done upstream, before data ever
reaches AMPL, on the API side — see `../amplpy/core-workflow-best-practices.md` for the
pandas/DataFrame-level equivalent.

## Prefer logical modeling constructs to their algebraic encodings

Where a business rule is naturally a logical statement, AMPL lets you write it close to
that natural form rather than manually deriving a big-M or indicator encoding — easier
to read, and easier to verify against the actual business requirement:

```ampl
subject to ShipmentDomain {p in PRODUCTS}:
   Shipment[p,w] == 0 or Shipment[p,w] >= min_shipment[w];
```

**Flag for verification**: the style guide's other example uses an implication operator
written `==>` (`ContainerFinish[c,t] = 1 ==> ContainerMove[c,t+1] = 0;`). This construct
does not appear in the AMPL Reference Manual (Appendix A) as covered in
`12-syntax-quick-reference.md`, and was not encountered anywhere else in the raw
sources — it may be part of AMPL's solver-specific logical/constraint-programming
extensions (used with CP-capable solvers) rather than core AMPL syntax. Verify against
current AMPL documentation before relying on `==>` in a model; the safer, universally-
supported alternative is an explicit `or`/big-M encoding as shown in the first example.

## See also

- Formal syntax for `check`, comments, and declarations: `01-language-basics.md`,
  `12-syntax-quick-reference.md`
- The equivalent data-validation habit from the Python side: `../amplpy/core-workflow-best-practices.md`
- Worked, larger example models that follow (most of) these conventions:
  `../tutorials-and-examples/worked-example-tutorial.md`
