# Data Files and Formats

Derived from: ampl-book.md (ch. 9)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 9.

## The `data` command and data mode

```
ampl: data diet.dat;      # read one file
ampl: data;                # enter data mode at the interactive prompt; type statements directly
```

- Whitespace (space/tab/newline) is collapsed to a single separator; commas between values are
  ignored too — arrange data into whatever layout reads best, it has no semantic effect.
- Strings generally don't need quotes; quote them if they contain anything other than
  letters/digits/`_`/`.`/`+`/`-`, or if they look like a number (`"3e4"`), to disambiguate from
  an actual numeric value.
- A `data` command (or reading a data file to the end without a final `;`) puts AMPL in data
  mode; any statement not starting with a data keyword (`set`, `param`, `var`) — e.g.
  `display` — returns to model mode.
- **Each parameter/set may be assigned data only once** across all `data` commands in a
  session; a second assignment is flagged as a duplicate-data error unless you explicitly issue
  `reset data` or `update data` first (`08-commands-modifying-models-and-data.md`) — this is a
  safety feature, not a limitation to work around casually.

## Lists: one-dimensional

```ampl
set PROD := bands coils plate;
param rate := bands 200  coils 140  plate 160;
```

Duplicate set members, or a parameter value given twice for the same key, are rejected with an
error naming the offending entry.

## Lists: two-dimensional and the "template" mechanism

A pair-indexed set/parameter can be listed as full tuples, as bare pairs, or — most
compactly — sliced with a **template**: a tuple with one or more components replaced by `*`,
followed by the list of values to substitute for that `*`:

```ampl
set LINKS :=
   (GARY,*) DET LAN STL LAF
   (CLEV,*) FRA DET LAN WIN STL LAF
   (PITT,*) FRA WIN STL FRE ;
```

Set templates use parentheses (`(GARY,*)`); parameter templates use brackets (`[GARY,*]`) since
they mirror subscript notation. A template with no `*` at all (a literal tuple) needs no
following value list — it specifies only itself. A bare list of tuples with no template header
implicitly uses the all-`*` template (`(*,*)` / `[*,*]`).

```ampl
param cost :=
   [GARY,*] DET 14  LAN 11  STL 16  LAF 8
   [CLEV,*] FRA 27  DET 9   LAN 12  WIN 9  STL 26  LAF 17
   [PITT,*] FRA 24  WIN 13  STL 28  FRE 99 ;
```

A template applies to every item until the next template header (or end of statement). This
generalizes to any dimension: an n-dimensional template has n slots, each a literal value or
`*`; for higher-dimensional sets you may slice through **more than one** dimension at once by
using more than one `*` (values are then supplied in matching tuples, substituted left to
right):

```ampl
set ROUTES :=
   (CLEV,*,bands) FRA DET LAN STL LAF        # slice through 2nd dim only
   (*,FRA,*) CLEV bands  CLEV coils  PITT bands   # slice through 1st and 3rd dims -> pairs of values
   ;
```

## Combined set + parameter lists (avoid specifying membership twice)

```ampl
param: PROD: rate profit market :=
   bands  200  25  6000
   coils  140  30  4000
   plate  160  29  3500 ;
```

The header line is: `param`, `:`, optionally `<indexing-set-name>` `:`, then the list of
parameter names, then `:=`. This single statement defines `PROD`'s membership *and* all three
parameters' values in one place — the standard idiom once you have more than one or two
parameters sharing an index set. **The named set must not already have a value** (e.g. from a
model-level `=` definition) — combining only works when the set's membership is genuinely
supplied by this same statement.

Parameters listed together in one combined statement don't have to share the *same* index set —
only that their headers are compatible; e.g. `n_min`/`n_max` over different (overlapping) sets
`MINREQ`/`MAXREQ` can share one table, using `.` for cells that don't apply to one of the two:

```ampl
param:  n_min   n_max :=
   A     700    20000
   C     700        .    # C is not in MAXREQ -- dot means "no value here", not zero
   NA      .    50000    # NA is not in MINREQ
   CAL 16000    24000 ;
```

**`.` (a literal dot) always means "no value here"** — never use a blank (ignored as
whitespace, silently shifts columns) or `0` (a real, wrong, data value) to represent "not
applicable."

## Tables

```ampl
param cost: FRA DET LAN WIN STL FRE LAF :=
   GARY  39  14  11  14  16  82   8
   CLEV  27   9  12   9  26  95  17
   PITT  24  14  17  13  28  99  20 ;
```

- Row labels = first index, column labels = second index. `(tr)` after the name transposes
  (row = 2nd index, column = 1st) — handy when one index set is much larger than the other and
  you want the long one running down instead of across.
- Split a wide table across multiple physical lines (row continues where it left off) or into
  several column-blocks (each new block starts with a bare `:` before the next column-header
  line) — purely a readability choice, `.`-marked gaps and blank cells behave the same either
  way.
- `.` marks a combination that's excluded from a sparse index set (e.g. `LINKS` doesn't contain
  that pair) — same convention as in lists.
- A **set** itself can be given in table form using `+`/`-` in place of values (`+` = member,
  `-` = not a member) — useful for eyeballing a sparse 2-D relationship at a glance.
- **Higher-dimensional / multi-index tables**: put more than one index on the left and/or top
  by stacking header lines, each preceded by `:`, with `:=` only after the last header line; or
  slice one/more dimensions out via a `[*,:,*]`-style template (`*` = goes on the left, `:` =
  goes on top) combined with a table. Index order in the underlying tuple is always preserved
  regardless of how you arrange left/top/transpose.

There is no single "correct" layout for multi-dimensional data — pick whichever slicing axis
groups the data most naturally for your source (e.g. one table per product is often more
natural than one table per week).

## Default values

```ampl
param cost default 9999 := ...;      # in the data: any un-supplied or dot-marked entry becomes 9999
param supply default 1;              # no explicit values at all -- every member gets 1
```

vs. declaring the default in the **model**:

```ampl
param cost {ORIG,DEST,PROD} >= 0, default 9999;             # fixed default
param cost {ORIG,DEST,p in PROD} >= 0, default huge_cost[p]; # default computed from other data
```

Put `default` in the data file to avoid repeating one constant value many times; put it in the
model when the default should itself be a formula depending on other parameters (data-file
`default` values must be literal, not expressions).

## Indexed collections of sets

```ampl
set AREA {PROD};
...
set AREA[bands] := east north;
set AREA[coils] := east west export;
```

Every data-statement format available for ordinary sets works the same way with a subscript
added to the set name. To avoid a "missing data" warning for members you intentionally leave
empty, declare `set AREA {PROD} default {};` in the model.

## Initial values for variables

Any `param`-style data statement can target a variable's name instead (optionally spell it
`var` for clarity) to seed a starting value:

```ampl
var Trans: FRA DET LAN WIN STL FRE LAF :=
   GARY 100 100 800 100 100 500 200
   ...  ;
```

Mostly relevant for nonlinear solvers that benefit from a warm start (`nonlinear-programs.md`);
LP solvers rarely use it. After `solve`, the *original* initial value is still retrievable via a
suffix (`Var.init`), documented in `12-syntax-quick-reference.md`.

## The `read` command (unformatted data)

For a file that contains **only bare values** (no names, colons, or `:=`), in a known fixed
order:

```ampl
read T, {t in 1..T} avail[t] < week_data.txt;
```

General form: `read item-list < filename;`, where each item is a parameter reference, an
indexing expression applied to one parameter reference, or an indexing expression applied to a
parenthesized list of items (nestable) — this lets you match whatever order the source file's
columns/records actually come in, including "one record per index value, several fields per
record":

```ampl
read {p in PROD} (prodcost[p], invcost[p], {t in 1..T} revenue[p,t]) < cost_data;
```

- Reading indexed over a set of strings uses AMPL's *internal* member order for that set —
  matches data-statement order if the set came from one, but for predictable behavior across
  read/data sources, declare the set `ordered` (`03-sets-and-indexing.md`).
- Omit `< filename` to read from the current input stream (useful for interactive prompting, or
  scripts); filename `-` means process stdin explicitly.
- `read` supplements rather than replaces `data`/database input — the sets being indexed into
  must already exist. It's most useful for large machine-generated files, or where the source
  format is closer to "a column of raw numbers" than to AMPL's own data-statement syntax — if
  you're pulling from an actual database or dataframe, prefer the table-handler / `amplpy`
  routes instead (`07-database-access-and-tables.md`, `../amplpy/`).

## See also

- Sets and compound-set concepts these formats encode: `03-sets-and-indexing.md`
- Reading data from relational databases / spreadsheets / CSV via table handlers:
  `07-database-access-and-tables.md`
- Changing data after it's loaded (`let`, `reset data`, `update data`):
  `08-commands-modifying-models-and-data.md`
- Full formal grammar for data statements: Appendix A.12, summarized in `12-syntax-quick-reference.md`
