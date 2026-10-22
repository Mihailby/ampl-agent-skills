# Database Access (`table` declarations)

Derived from: ampl-book.md (ch. 10)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 10.

Note: this is AMPL's native `table` statement for connecting to relational sources (ODBC/
Access/Excel, or AMPL's built-in text/binary table formats) directly from a model or script.
If you're working from Python, `amplpy` typically reads a pandas DataFrame straight into AMPL
instead of going through `table` declarations — see `../amplpy/core-workflow-best-practices.md`.
This file covers the underlying `table`/`read table`/`write table` mechanism, which is still
useful in pure-AMPL scripts or when the DB connection needs to live inside the model itself.

## Mental model: relational tables ↔ AMPL entities

Any AMPL set/parameter/variable indexed over the *same* set corresponds naturally to one row
per set member, with one database column per AMPL entity, and one **key column** per dimension
of the index set. A `table` declaration is just a mapping between the two; `read table` and
`write table` are the commands that actually move data using that mapping.

```ampl
set FOOD;
param cost {FOOD} > 0;
param f_min {FOOD} >= 0;
param f_max {j in FOOD} >= f_min[j];
```

↔

```
FOOD    cost   f_min   f_max
BEEF    3.19   2       10
...
```

Two-dimensional indices need two key columns, and so on — the pattern generalizes directly to
any dimension.

## Declaring and reading a table

```ampl
table Foods IN "ODBC" "diet.mdb": FOOD <- [FOOD], cost, f_min, f_max;
table Foods IN: FOOD <- [FOOD], cost, f_min, f_max;      # AMPL's built-in text/binary handler
...
read table Foods;
```

- `table-name` (`Foods`) is the handle used within AMPL; it need not match the external table's
  own name (given as a separate string when it differs).
- `IN` at the top level means "read-only" for every column by default; `OUT` means write-only;
  omit both (or use `INOUT`) for a table that reads some columns and writes others.
- The bracketed key-spec's `<-` means "copy key-column values into this AMPL set":
  `FOOD <- [FOOD]` reads column `FOOD` both to populate set `FOOD` and to serve as the row key
  for the other parameters. If you only want the key columns as subscripts (the set already
  exists / isn't stored this way), drop the `set-name <-` part: `[ORIG, DEST], cost;`.
- `read table Foods;` performs the actual transfer; the declaration only defines the mapping.
  Column-to-parameter names don't have to match the underlying database if you use `~`
  (see below); subscripts aren't checked for validity until the parameter is first used
  (typically at `solve`).
- **Scalar (unindexed) parameter** from a one-row table: use an empty key-spec, `[]`.

## Naming/reordering mismatches: the `~` operator and key-column dummy indices

```ampl
table Foods IN: [FOOD], cost, f_min ~ lowerlim, f_max ~ upperlim;   # AMPL name ~ external column name
table SteelProd IN: [p ~ PROD, t ~ TIME],                            # give the key columns dummy indices
   market[p,t+1] ~ market,           # renumber periods (external counts from 0)
   revenue[t,p] ~ revenue;           # different subscript ORDER than the AMPL parameter
```

`~` is required whenever the natural AMPL expression (e.g. `revenue[t,p]`, or anything with a
`.` like `Diet.lslack`) isn't itself a legal database column name — most database software
rejects punctuation in column names, so any suffixed or subscripted expression on the AMPL side
needs an explicit `~ column_name` on the output side.

Any **assignable expression** can appear as a data-spec on read, not just parameters: variable
names (to seed initial/warm-start values), constraint names (initial duals, for solvers like
MINOS that use them), and suffixed variables/constraints (e.g. `Buy.priority ~ prior` to feed
MIP branching priorities to CPLEX).

## Writing results back out

```ampl
table Foods OUT: [FoodName], f_min, Buy, f_max;                  # rows inferred from the data-specs' index sets
table Foods OUT: FOOD -> [FoodName], f_min, Buy, f_max;          # rows inferred explicitly from the -> set
```

Two ways to determine which rows get written:

1. **From the data-specs** (`[key-cols], expr, expr, ...`): AMPL takes the union of the index
   sets implied by the listed expressions — mirrors `display` semantics. If the expressions are
   indexed over different-but-related sets (e.g. `Make`/`Sell` over `{PROD,1..T}` vs. `Inv` over
   `{PROD,0..T}`), you get one row per member of the *union*, with `.` in cells that don't
   apply.
2. **From an explicit key-spec** (`set-spec -> [key-cols]`): the row set is stated up front —
   `{t in 1..T} -> [TIME], ...` writes exactly `T` rows, no more, no fewer, regardless of what
   the data-specs are indexed over. Dummy indices can be introduced either in the `set-spec`
   (`{p in PROD, t in 1..T} -> [PROD,TIME]`) or attached to the key-col-specs
   (`{PROD,1..T} -> [p ~ PROD, t ~ TIME]`) — if both are given, the key-col-spec index wins.

General expressions (not just bare parameter/variable names) are allowed in OUT data-specs,
including `sum`-based aggregates and suffix expressions — always pair a non-trivial expression
with `~ column_name`:

```ampl
table SteelTotal OUT: [TIME],
   {t in 1..T} (sum {p in PROD} Make[p,t] ~ Made, sum {p in PROD} Sell[p,t] ~ Sold);
```

**Overwrite semantics**: if every data-spec in the declaration is `OUT`, the whole external
table/file is recreated; if any column is `IN` or `INOUT`, `write table` only touches the `OUT`
columns and leaves the rest of the external table alone (existing rows updated in place by key,
new rows appended).

## Reading and writing the same external table

Either use two separate `table` declarations pointed at the same external file/table name (one
`IN`, one with a mix of `IN`/`OUT` columns), or one combined declaration with per-column
`IN`/`OUT`/`INOUT` status and `<->` on the key-spec:

```ampl
table Foods "ODBC" "diet1.mdb" "Diet":
   FOOD <- [FoodName], cost IN, f_min IN, Buy OUT, f_max IN;
```

`read table Foods` then touches only the `IN`/`INOUT` columns; `write table Foods` touches only
`OUT`/`INOUT`.

## Indexed collections of tables and columns

A `table` declaration itself can be indexed (one external table/file per set member — e.g. one
sensitivity-analysis table per food, generated in a loop), and a single data-spec can expand
into a collection of columns via `{indexing-expr} < data-spec, ... >` (e.g. writing a 2-D
parameter or variable to a spreadsheet with one column per member of the second index,
column names built with string concatenation `&`):

```ampl
table DietSens {j in FOOD} OUT "ODBC" "DietSens.mdb" ("Sens" & j): [Food], f_min, Buy, f_max;
table dietAmts IN "ODBC" "Diet2D.xls": [i ~ NUTR], {j in FOOD} <amt[i,j] ~ (j)>;
```

`(expr)` — parenthesized — forces AMPL to evaluate it as a *string expression* for a column
name, rather than reading it as a literal column-name token.

## Table handlers

`table` declarations hand off the actual I/O to a **table handler**, selected by the first
string in the string-list after `IN`/`OUT`:

- **`"ODBC"`** — AMPL's standard handler, Windows-only, talks to Access (`.mdb`), Excel
  (`.xls`), or any ODBC-registered source. String-list form:
  `"ODBC" "connection-spec" "external-table-spec"? "verbose"?`. The third string can be a literal
  table/range name, or `"SQL=<query>"` to read from an ad-hoc `SELECT` (all columns must then be
  `IN` — you can't write to a query result).
- **Built-in text/binary handlers** — no first string needed; a filename ending `.tab` (plain
  text, human-readable/diffable) or `.bit` (binary, faster) selects the format. Useful for
  testing/demos and for tables you want to inspect by eye.
- Other handlers (vendor- or solver-supplied) are loaded the same way via their own first-string
  keyword; check the specific handler's documentation for its string-list format.

## See also

- Non-database data statement formats (lists, tables, templates): `06-data-files-and-formats.md`
- Reading raw unformatted value streams: `06-data-files-and-formats.md` (`read` command)
- Scripting a loop that reads/solves/writes per scenario: `10-scripting-and-control-flow.md`
- The Python-native equivalent (DataFrame ↔ AMPL): `../amplpy/core-workflow-best-practices.md`
