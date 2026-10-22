# Sets and Indexing

Derived from: ampl-book.md (ch. 5-6)

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapters 5-6.

Sets are the most fundamental component of an AMPL model — nearly every parameter, variable,
and constraint is indexed over one, and set indexing is what lets a short, concise model
describe an arbitrarily large mathematical program.

## Table of contents

- Simple (unordered) sets
- Sets of numbers, `..` ranges
- Set operators: `union`, `inter`, `diff`, `symdiff`
- Membership: `in`, `within`, `card`
- Indexing expressions (the `{...}` construct), scope, conditions
- Ordered sets: `first`, `last`, `next`, `prev`, `ord`, `member`
- Predefined/infinite sets and interval expressions
- Compound sets: pairs, triples, tuples
- Subsets and "slices" of compound sets
- `cross`, `setof`, iterated `union`/`inter`
- Indexed collections of sets

## Simple (unordered) sets

```ampl
set PROD;                                    # membership supplied via data
set PROD = {"bands", "coils", "plate"};      # literal set, fixed in the model
```

- Members are character strings (quoted with `'...'` or `"..."`) and/or numbers. Case matters:
  `"fish"`, `"Fish"`, `"FISH"` are distinct.
  A number (`1`) and the corresponding string (`"1"`) are distinct, but different numeric
  representations of the same value (`100` and `1E+2`) are the same member.
- `{}` is the empty set; `{"bands"}` (a set of one) is distinct from the string `"bands"` itself.
- Use `=` for a genuinely fixed membership; use `default` instead of `=` if the value should be
  overridable by later data/assignment — parallels the same distinction for parameters
  (`04-parameters-and-expressions.md`).
- **Reserved words**: names built from letters/digits/`_` that aren't a valid number; a fixed
  list is reserved outright (`sum`, `in`, ...; see `12-syntax-quick-reference.md`), but many
  more (like `prod`) are only *predefined* and can be reused as your own identifiers if you
  want (`set prod;` is legal).

## Sets of numbers

```ampl
{1,2,3,4,5,6}        # literal
1..6                 # equivalent, more concise
1990 .. 2020 by 5     # {1990,1995,...,2020} -- explicit step
```

Prefer driving `..` ranges from parameters rather than literals so the model doesn't need
editing when the horizon changes:

```ampl
param start integer;
param end > start integer;
param interval > 0 integer;
set YEARS = start .. end by interval;
```

Numeric set members are stored as floating point — avoid relying on exact equality for
fractional set members (`-5/3 .. 5/3 by 1/3` will *not* contain exactly `0` or `1` due to
rounding). Integers are exact up to about 2^53, so integer-valued sets are safe.

## Set operators

```ampl
A union B      # in A or B
A inter B      # in both
A diff B       # in A but not B
A symdiff B    # in A or B but not both
```

Operands can be arbitrary set expressions, including parenthesized combinations:
`(Y1 union {2025,2035,2045}) diff Y2`. `union`/`diff`/`symdiff` share one precedence level,
below `inter`; `A union B inter C diff D` parses as `(A union (B inter C)) diff D`. Full table:
`12-syntax-quick-reference.md`.

Use set operators in the assignment phrase of a declaration to *derive* one set from others
instead of asking the data to supply redundant information:

```ampl
set MINREQ;  set MAXREQ;                 # supplied in data
set NUTR = MINREQ union MAXREQ;          # derived -- never inconsistent with the parts
```

As a general modeling principle: choose a minimal set of independent sets to appear in the
data, and define every other relevant set as an expression in the model. This makes
inconsistent data structurally impossible instead of merely undesirable.

## Membership: `in`, `within`, `card`

```ampl
"B2" in NUTR            # true if "B2" is a member of NUTR         (member, ∈)
MINREQ within NUTR      # true if MINREQ is a subset of NUTR       (set,   ⊆)
not in / not within     # negations
card(NUTR)               # cardinality (number of members)
```

`within` can also appear directly in a set's own declaration to constrain its data:

```ampl
set NUTR;
set MAXREQ within NUTR;   # AMPL rejects data that puts a non-NUTR member into MAXREQ
```

This is cheap, load-bearing input validation — prefer it over discovering the mistake later
as a stray `KeyError`-style failure deep in solve output.

## Indexing expressions

The `{...}` construct is AMPL's rendering of "for each ... in ..." from algebraic notation. It
appears in declarations, in `sum`/other iterated operators, and anywhere else a set is valid.

```ampl
param rate {PROD} > 0;                       # no dummy index needed
param f_max {j in FOOD} >= f_min[j];          # dummy index j, used in the bound expression
subject to Time {t in 1..T}:
   sum {p in PROD} (1/rate[p]) * Make[p,t] <= avail[t];
```

**Scope**: a dummy index is defined only within the indexing expression that introduces it —
for a `subject to` declaration, that's the whole statement; for a `sum`, just its summand. Two
different index names are required when one loop is nested inside another over the same set:

```ampl
subject to Food_Ratio {j in FOOD}:
   Buy[j] >= min_frac[j] * sum {jj in FOOD} Buy[jj];   # jj, not j, in the inner sum
```

A common, purely conventional practice: reserve certain letters for certain sets throughout a
model (`i` always over origins, `j` always over destinations) to aid readability — not an AMPL
requirement.

**Conditions**: append `: <logical-condition>` to filter membership:

```ampl
{j in FOOD: f_max[j] - f_min[j] < 1}
{i in NUTR: i in MAXREQ or n_min[i] > 0}
```

A conditioned indexing expression *is* a set expression and can be used anywhere a set can,
including named as its own set:

```ampl
set NUTREQ = {i in NUTR: i in MAXREQ or n_min[i] > 0};
```

Interactive commands (`display`, etc.) may reference variable/constraint values in a condition
(e.g. `display {i in MINREQ: Diet_Min[i].slack = 0};`); inside a model, only sets, parameters,
and dummy indices are allowed in a condition — not variables or constraint suffixes.

Indexing over an empty set is not an error — AMPL simply generates nothing for that component,
and a `sum` over an empty set is `0`.

## Ordered sets

Add `ordered` (or `circular`, which wraps the last member back to the first) to a set
declaration to fix the order in which members were given as *the* order AMPL uses:

```ampl
set WEEKS ordered;
```

Functions available only on ordered sets:

| Function | Meaning |
|---|---|
| `first(S)`, `last(S)` | first/last member |
| `member(k, S)` | the k-th member |
| `next(t, S)`, `prev(t, S)` | member immediately after/before `t` |
| `next(t, S, k)`, `prev(t, S, k)` | k-th member after/before `t` |
| `nextw`, `prevw` | wraparound versions (treat the set as circular) |
| `ord(t, S)` | numeric position of `t` (error if not a member) |
| `ord0(t, S)` | same, but returns `0` instead of erroring |

If the first argument is a dummy index already known to run over an ordered set, the set
argument can be dropped (`prev(t)` instead of `prev(t, WEEKS)`).

This is the idiomatic replacement for "the previous period" logic once periods are meaningful
labels (`"27sep"`) rather than plain integers (where `t-1` sufficed):

```ampl
subject to Balance0 {p in PROD}:
   Make[p,first(WEEKS)] + inv0[p] = Sell[p,first(WEEKS)] + Inv[p,first(WEEKS)];
subject to Balance {p in PROD, t in WEEKS: ord(t) > 1}:
   Make[p,t] + Inv[p,prev(t)] = Sell[p,t] + Inv[p,t];
```

Calling `prev()` on the first member of a non-circular ordered set is a runtime error — always
special-case the first period (as above) or filter it out (`t in WEEKS diff {first(WEEKS)}`,
or `ord(t) > 1`).

`diff` preserves the ordering of its left operand; `union`/`inter`/`symdiff` do not — their
result is treated as unordered.

To make a *subset* inherit an existing set's ordering: `ordered by <set>` (or
`ordered by <set> reversed` / `circular by ...`):

```ampl
set ALL_WEEKS ordered;
param T > 0 integer;
set WEEKS = {t in ALL_WEEKS: ord(t) <= T} ordered by ALL_WEEKS;
```

## Predefined and infinite sets, interval expressions

Interval shorthand for declaring bounds (not usable as an indexing set — too large/infinite):

```ampl
param rate {PROD} in interval (0,maxrate];    # equivalent to: > 0, <= maxrate
param rate {PROD} in (0, Infinity];           # "interval" keyword is optional; equivalent to > 0
```

`Infinity` / `-Infinity` are predefined unbounded endpoints. `Reals` and `Integers` are the
predefined infinite sets of all floats / all integers. `ASCII`, `EBCDIC`, `Display` are
predefined orderings over the universe of strings/numbers — useful for forcing alphabetical (or
display-consistent) ordering of a set regardless of data order:

```ampl
set PROD ordered by ASCII;
```

## Compound sets: pairs, triples, tuples

```ampl
("PITT","STL")             # ordered pair -- order matters
(3, 101)
{("PITT","STL"), ("PITT","FRE")}     # literal set of pairs
```

All members of one set must share a dimension (can't mix pairs and triples). The set of *all*
pairs from two sets is written `{ORIG, DEST}` or, with dummy indices, `{i in ORIG, j in DEST}`;
referencing a component then requires the full subscript list: `cost[i,j]`, `Trans[i,j,p]`.

Naming a pairs-set explicitly emphasizes its role (e.g. "links" between cities):

```ampl
set LINKS = {ORIG,DEST};
...
sum {(i,j) in LINKS} cost[i,j] * Trans[i,j];   # dummy indices must be written as a tuple (i,j)
```

## Subsets and slices of compound sets

When not every combination is valid/meaningful, declare a genuine subset instead of the full
cross product — this also documents intent and lets AMPL validate the data:

```ampl
set LINKS within {ORIG,DEST};     # (bare "set LINKS dimen 2;" works too, but is less clear)
param cost {LINKS} >= 0;
var Trans {LINKS} >= 0;
```

The key technique is the **slice**: when one or more tuple components are already bound (e.g.
by an outer indexing expression), re-using the same set expression automatically restricts to
the matching "row"/"column":

```ampl
subject to Supply {i in ORIG}:
   sum {(i,j) in LINKS} Trans[i,j] = supply[i];   # {(i,j) in LINKS} = "all j linked to this i"
subject to Demand {j in DEST}:
   sum {(i,j) in LINKS} Trans[i,j] = demand[j];   # here it means "all i linked to this j"
```

This is both more concise *and* more efficient than the spelled-out alternative
(`sum {j in DEST: (i,j) in LINKS} Trans[i,j]`), because AMPL can walk directly to the matching
tuples instead of generating the full cross product and filtering it. **This matters at scale**:
with high-dimensional sets, prefer the slice form —

```ampl
subject to Supply_All {i in ORIG}:
   sum {(i,j,p) in ROUTES} Trans[i,j,p] <= supply_all[i];              # efficient
# vs.
subject to Supply_All {i in ORIG}:
   sum {j in DEST, p in PROD: (i,j,p) in ROUTES} Trans[i,j,p] <= supply_all[i]; # generates the full DEST×PROD cross product first
```

Declaring a set like `ROUTES within {ORIG,DEST,PROD}` does *not* force AMPL to materialize the
full (possibly huge) cross product — only the actual tuples given in the data are stored;
the `within` phrase is a validation constraint, not a memory cost, as long as `ROUTES` itself
stays reasonably sized.

## `cross`, `setof`, iterated set operators

```ampl
ORIG cross DEST                      # same set as {ORIG, DEST}
set PROD = setof {(i,j,p) in ROUTES} p;          # project a compound set down to one column
set LINKS = setof {(i,j,p) in ROUTES} (i,j);     # ...or down to a pair; duplicates auto-dropped
```

Iterated `union {p in PROD} orig[p]` / `inter {p in PROD} orig[p]` work like iterated `sum`, but
combine sets instead of numbers.

Two different dummy indices over the *same* set, with a filter, is the pattern for "all distinct
pairs" or "triangular" combinations:

```ampl
set TRANSF = {i1 in ORIG, i2 in ORIG: i1 <> i2};                       # all unordered? no -- both directions
set ORIG ordered;
set TRANSF = {i1 in ORIG, i2 in ORIG: ord(i1) < ord(i2)};              # true triangular, no reverse-duplicates
```

When one indexing phrase's dummy index is used in a later phrase in the same expression
(comma-separated phrases evaluate left to right), you get relative/derived ranges for free:

```ampl
subject to Too_Old {p in PROD, t in 1..T, a in t+1..A}: Inv[p,t,a] = 0;
```

## Indexed collections of sets

A *set itself* can be declared as an indexed collection, one set per index value — appropriate
when there's a natural hierarchy (e.g. each product has its own set of market areas), as opposed
to a symmetric relationship (origins/destinations), which is more natural as a set of tuples.

```ampl
set PROD;
set AREA {PROD};                      # AREA[p] is a distinct set for each product p
param market {p in PROD, AREA[p], 1..T} >= 0;
var Sell {p in PROD, a in AREA[p], t in 1..T} >= 0, <= market[p,a,t];
```

Data supplies each `AREA[p]` like an ordinary set:

```ampl
set AREA[bands] := east north;
set AREA[coils] := east west export;
```

Indexed collections of sets combine with `within` and set operators just like ordinary ones,
including derived collections:

```ampl
set orig {PROD} within ORIG;
set dest {PROD} within DEST;
set links {p in PROD} = orig[p] cross dest[p];    # a compound set, itself indexed by product
```

**Choosing between a set of tuples and an indexed collection of sets**: they can express the
same information, but pick based on the relationship's shape. Hierarchical/asymmetric
relationships (a product "owns" its areas; you never need "all products sold in a given area")
read more naturally as an indexed collection. Symmetric relationships (an origin-destination
link is just as naturally viewed from either side) read more naturally as a set of tuples.

## See also

- Parameters, expressions, and `check` statements: `04-parameters-and-expressions.md`
- Variables/objectives/constraints (`var`, `subject to` mechanics): `05-variables-objectives-constraints.md`
- How compound-set data is actually laid out in `.dat` files: `06-data-files-and-formats.md`
- Applied versions of these patterns: `../modeling-patterns/classic-lp-patterns.md`
