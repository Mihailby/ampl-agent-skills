# Command Scripts and Control Flow

Derived from: ampl-book.md (ch. 13), dynamic-source-updates-and-automated-data-sets.md

Source: *AMPL: A Modeling Language for Mathematical Programming*, 2nd ed., Chapter 13;
`dynamic-source-updates-and-automated-data-sets.md`.

A **script** is a sequence of AMPL commands saved to a file for reuse — the natural next step
once you're re-typing the same `model`/`data`/`solve`/`display` sequence for different data.
Scripts can use for/repeat/if control flow to become small programs in AMPL's own command
language.

## Running scripts: `include` vs. `commands`

```
ampl: include myfile.run;     # inserted in place, mode-preserving, no trailing ; required
ampl: commands myfile.run;    # a true statement (needs ;), safer inside for/repeat/if bodies
```

- `model file;` / `data file;` are really just `include` with the mode forced to model/data
  first, then reverted afterward — you can freely use `model`/`data`/`include` interchangeably
  to read any file; only the initial mode differs.
- **Prefer `commands` over `include` inside scripts**, especially inside loop/conditional
  bodies — `include` can interact unexpectedly with the parsing of `for`/`repeat`/`if`.
- A common pattern: one `.run` "driver" file does `model ...; data ...; solve; option ...;
  display ...;` in sequence — read it once with `include`/`commands` to set up and solve a
  whole scenario in one command.

## `for`: iterate over a set

```ampl
for {1..4} commands sensitivity.run;             # repeat a fixed number of times
for {a in AVAIL3} {                              # iterate with the member bound to a name
   let avail[3] := a;
   solve;
   let avail3_obj[a] := Total_Profit;
   let avail3_dual[a] := Time[3].dual;
}
```

- The expression after `for` is any indexing expression — a literal range, a declared set, or a
  filtered/derived set expression.
- Loop order follows the set's ordering if it's `ordered`/`circular`/numeric; for an unordered
  set, AMPL picks *some* consistent order and reuses it every pass — reliable enough to build a
  table column-by-column across nested loops (see the worked table-building example below), but
  don't rely on it matching your `.dat` file's literal order — declare the set `ordered` if that
  matters (`03-sets-and-indexing.md`).
- Building a formatted, arbitrary-shape table (rows *and* columns depend on the data) requires a
  **nested for loop with `printf`**, not a single `printf` — a single `printf`'s format string
  has a fixed number of conversion specs, so it can't handle a variable number of columns:
  ```ampl
  printf "\nSALES";
  printf {p in PROD}: "%14s   ", p;
  printf "\n";
  for {t in 1..T} {
     printf "week %d", t;
     for {p in PROD} { printf "%9d", Sell[p,t]; printf "%7.1f%%", 100*Sell[p,t]/market[p,t]; }
     printf "\n";
  }
  ```

## `repeat`: iterate until/while a condition holds

```ampl
repeat while <cond> { ... };     # test BEFORE each pass (may run zero times)
repeat until <cond> { ... };
repeat { ... } while <cond>;     # test AFTER each pass (always runs at least once)
repeat { ... } until <cond>;
repeat { ... }                   # no condition -- must self-terminate via break
```

Typical use: sensitivity sweeps where you don't know the stopping point in advance (e.g. "keep
raising this resource limit until its dual value hits zero"):

```ampl
set AVAIL3 default {};                 # "default", not "=" -- must stay let-able as the loop runs
param avail3_obj {AVAIL3};
param avail3_dual {AVAIL3};
repeat {
   solve;
   let AVAIL3 := AVAIL3 union {avail[3]};
   let avail3_obj[avail[3]] := Total_Profit;
   let avail3_dual[avail[3]] := Time[3].dual;
   let avail[3] := avail[3] + 5;
} until Time[3].dual = 0;
```

**Floating-point equality gotcha**: `option display_precision` rounds only what's *shown*, not
the stored value — a value that displays as `1400` may actually be `1399.9999999999995`. Loop
conditions like `until Time[3].dual = 0` can misbehave for this reason. Fixes: compare with a
tolerance (`until Time[3].dual <= 0.0000001`), or set `option solution_precision 10;` (or
`solution_round`) **before** the first `solve` so stored values themselves get cleaned up
(`09-display-and-output-commands.md`).

## `if` / `then` / `else`

```ampl
if Make["coils",2] < 1500 then printf "under 1500\n";
if <cond> then { stmt; stmt; } else { stmt; }
if a then ...; else if b then ...; else ...;        # else always binds to the nearest preceding if
```

Same condition syntax as the `if-then-else` *expression* (`04-parameters-and-expressions.md`),
but here it's a *statement* controlling which commands run, not a value.

## `break` / `continue`, and naming loops

- `continue` — skip the rest of the current pass, go to the next pass's condition check.
- `break` — exit the loop entirely, resume after it.
- Both apply to the **innermost** enclosing loop by default. To break/continue an *outer* loop
  from inside a nested one, name the outer loop and reference that name:
  ```ampl
  repeat sens_loop {
     for {t in 1..T}
        if Time[t].dual < 2700 then break sens_loop;   # exits the REPEAT, not just the FOR
     ...
  }
  ```

## Stepping through a script (debugging)

```
ampl: option single_step 1;
ampl: commands myfile.run;
<2>ampl: step        # execute exactly the next command, show what's coming next
<2>ampl: step 3       # execute the next 3
<2>ampl: next         # like step, but treats a whole for/repeat/if as ONE step (doesn't dive in)
<2>ampl: skip [n]     # skip instead of executing
```

You can type ordinary commands (e.g. `display`) at any `<N>ampl:` breakpoint prompt to inspect
state mid-script before continuing.

## String functions (for building names, filenames, option values dynamically)

| Function | Purpose |
|---|---|
| `s1 & s2` | concatenate (numbers auto-convert to string, full precision — see float caveat below) |
| `length(s)` | character count |
| `match(s, pattern)` | first position of `pattern` in `s` (0 if absent); `pattern` may be a regex |
| `substr(s, start[, len])` | substring |
| `sub(s, pattern, repl)` / `gsub(...)` | replace first / all regex matches |
| `sprintf(fmt, args...)` | like `printf`, but returns the formatted string instead of printing it |

```ampl
set NUTR_FOOD := setof {i in NUTR, j in FOOD} i & "_" & j;   # build composite keys
set WEEKS ordered = setof {t in 1..T} "WEEK" & t;             # generate labels
```

**Float-to-string gotcha**: `&` converts numbers to strings at full precision, which for
non-exact binary fractions can produce surprises (`"WEEK" & 0.1*t` may yield `WEEK0.30000000000000004`
instead of the expected `WEEK0.3`, because `0.1*3` isn't exactly representable). Use
`sprintf("WEEK%3.1f", 0.1*t)` for controlled formatting instead of relying on `&`'s default
conversion.

**Splitting a composite key back into parts** — combine `match`/`substr` (or a `for` loop) to
decompose a flattened `"NUTRIENT_FOOD"`-style key back into two sets/parameters when data
arrives in a non-standard flattened form:
```ampl
set NUTR = setof {ij in NUTR_FOOD} substr(ij, 1, match(ij,"_")-1);
set FOOD = setof {ij in NUTR_FOOD} substr(ij, match(ij,"_")+1);
param amt {i in NUTR, j in FOOD} = amt_nutr[i & "_" & j];
```

**String expressions as filenames / option values / db identifiers**: allowed anywhere a
literal string could go (`data`, `>`/`>>` redirection targets, `option` values, `table`
string-lists) — wrap the expression in parentheses to signal it's computed, not literal:
```ampl
for {j in 1..3} {
   data ("diet" & j & ".dat");
   solve;
   display Buy >("diet" & j & ".out");
}
```

## Automatically-updated introspection sets

Beyond the `_var`/`_con`/`_obj` generic-synonym families (`09-display-and-output-commands.md`),
AMPL maintains always-current sets of every declared component's *name*, handy for
metaprogramming/generic scripts that shouldn't hard-code component names:

```ampl
_PARS     # set of all declared param names
_SETS     # set of all declared set names
_VARS     # set of all declared variable names
_CONS     # set of all declared constraint names
_OBJS     # set of all declared objective names
_PROBS    # set of all declared problem (named-problem) names
_ENVS     # set of all declared environment names
_FUNCS    # set of all declared (user-defined/imported) functions
```

## See also

- The `let`/`fix`/`drop` commands scripts most commonly call in a loop body:
  `08-commands-modifying-models-and-data.md`
- `display`/`print`/`printf` formatting used for script output: `09-display-and-output-commands.md`
- Full string-function reference (regex syntax details) and full script-statement grammar:
  `12-syntax-quick-reference.md`
- A worked, narrative walkthrough combining scripting with amplpy-side automation:
  `../tutorials-and-examples/cross-language-api-equivalents.md`
