# Tools, Troubleshooting, and Reformulation

Derived from: mp-tools-and-details.md, mp-troubleshooting.md

Source: `mp-tools-and-details.md` (`https://mp.ampl.com/modeling-tools.html`),
`mp-troubleshooting.md` (`https://mp.ampl.com/modeling-troublesh.html`).

This file covers how to look inside what MP actually did to a model — what form each
constraint reached the solver in, how to inspect and control that, how to sanity-check the
solution that came back — and what to try when a model solves too slowly, inaccurately, or
not at all.

## Flat constraints vs. expression trees

Some solvers require each expression to arrive as a separate *flat constraint*, with an
auxiliary variable standing in for the expression's result:

```
aux_var = max(x, y);
```

Others (or the same solver, for some construct types) accept the alternative: an
*expression tree*, i.e. a complete formula passed as a single unit —

```ampl
s.t. NLConstraint1: 2*x + 4*exp(16 - 2*sin(x + y^2)) <= 19;
```

with `exp(16 - 2*sin(x + y^2))` submitted intact rather than broken into auxiliary pieces.
Expression trees generally give better performance and numerical precision when the
solver supports them — SCIP 9.1.1 and Gurobi 12 are current examples of MP solvers that
do.

### Acceptance options (`acc:...`)

Control which form MP uses, per constraint/expression type, or globally:

- `acc:_all=2` — declare everything natively accepted (disables all automatic
  reformulation; useful to test manual modeling of high-level constructs).
- `acc:_all=0` — force full linearization everywhere.
- `acc:<name>=2` (e.g. `acc:alldiff=2`) — declare this specific construct natively
  accepted, disabling its reformulation.
- `acc:<name>=0` (e.g. `acc:or=0`) — force MP's reformulation for this construct even if
  the solver natively accepts it.
- `acc:_expr` — uniformly controls whether expressions are passed as trees (`1`, default —
  passed if natively supported, per the individual `acc:...` options) or as flat
  constraints (`0`).

A single `acc:<name>` option can take a finer-grained 0–4 scale (example for `acc:sin`,
default 4): `0` not accepted, automatic redefinition attempted; `1` accepted as a
constraint but redefinition preferred where possible; `2` accepted as a constraint and
preferred; `3` accepted as an expression but redefinition preferred where possible; `4`
accepted as an expression and preferred. `acc:_expr=1` corresponds to value 4 in the
per-construct scale; `acc:_expr=0` corresponds to value 2 or 0.

To see which constructs a given solver natively accepts, either run it with the `-=acc`
switch (e.g. `gurobi -=acc`, which lists every `acc:`-prefixed option) or check that
solver's option page at AMPL Development (`https://dev.ampl.com/solvers/index.html`).

### Full constraint list (`-c` switch)

Running a solver executable with `-c` lists every constraint type the MP model converter
knows about, including internal ones — useful for reading solution-check warnings (below)
or Reformulation Explorer output. Abbreviated reference (ID → full name):

| ID | Full name | ID | Full name |
|---|---|---|---|
| `abs` | AbsConstraint | `lin(le/eq/ge)` | Linear constraint |
| `acos`/`acosh` | Acos/AcoshConstraint | `linrange` | Linear range constraint |
| `alldiff` | AllDiffConstraint | `linfunccon` | LinearFunctionalConstraint |
| `and` | AndConstraint | `log`/`loga` | Log/LogAConstraint |
| `asin`/`asinh` | Asin/AsinhConstraint | `max`/`min` | Max/MinConstraint |
| `atan`/`atanh` | Atan/AtanhConstraint | `not` | NotConstraint |
| `compl`/`complquad` | Complementarity (linear/quadratic) | `numberofconst`/`numberofvar` | NumberofConst/VarConstraint |
| `condlin(lt/le/eq/ge/gt)` | Conditional linear constraint | `or` | OrConstraint |
| `condquad(lt/le/eq/ge/gt)` | Conditional quadratic constraint | `pl` | PLConstraint |
| `cos`/`cosh` | Cos/CoshConstraint | `pow` | PowConstraint |
| `count` | CountConstraint | `powercone` | PowerConeConstraint |
| `div` | DivConstraint | `quad(le/eq/ge)` | Quadratic constraint |
| `exp`/`expa` | Exp/ExpAConstraint | `quadrange` | Quadratic range constraint |
| `expcone` | ExponentialConeConstraint | `quadcone` | QuadraticConeConstraint |
| `geomcone` | GeometricConeConstraint | `quadfunccon` | QuadraticFunctionalConstraint |
| `ifthen` | IfThenConstraint | `rotatedquadcone` | RotatedQuadraticConeConstraint |
| `impl` | ImplicationConstraint | `sos1`/`sos2` | SOS1/SOS2Constraint |
| `ind(le/eq/ge)` | Linear indicator constraint | `sin`/`sinh` | Sin/SinhConstraint |
| `indquad(le/eq/ge)` | Quadratic indicator constraint | `tan`/`tanh` | Tan/TanhConstraint |

## Exporting the solver's model

To see exactly what MP and the solver did with a model, export it (and, where the solver
supports it, its presolved form):

```ampl
option gurobi_options 'writeprob=disj.lp writepresolved=disj_pre.lp';
option gurobi_auxfiles rc;     ## use var/con names in the export
solve;
```

Format is solver-dependent (determined by the file extension). For Mosek, an equivalent
example:

```ampl
option mosek_auxfiles rc;
option mosek_options 'writeprob=/tmp/ell.jtask'; solve;
```

## Reformulation Explorer

A tool for comparing the model as sent to an MP solver driver (from the NL file) against
the final model the underlying solver actually receives, including the intermediate
reformulation steps as a tree.

**Invocation.** Use it online at `https://ampl.com/streamlit/Reformulation_Explorer`, or
run it locally from the `ampl/mp` repository's `support/modelexplore` subfolder:

```
streamlit run modelexplore.py --server.maxUploadSize=1024
```

To produce the input file, solve with the `writegraph` option:

```ampl
option solver gurobi;
option gurobi_auxfiles rc;
option gurobi_options 'writegraph=model.jsonl lim:time=0';
solve;
```

(equivalently `ampl.solve(solver="gurobi", gurobi_options="writegraph=graph.jsonl")` via
amplpy, or `gurobi model.nl writegraph=reformulations.jsonl lim:time=0` from the command
line). Upload the resulting JSONL file to the Explorer; it displays the NL (source) and
solver's (destination) models side by side, with a text-pattern search over both and
download buttons for the displayed (sub)models. The NL model shown generally matches
AMPL's own `solexpand` output; the solver model matches what `tech:writeprob` would export.
Selecting *Reformulation tree* mode shows (and lets you download, for a separate JSON
viewer) the intermediate reformulation steps for a specific constraint.

Practical use: search for a constraint by name to see exactly which auxiliary variables
and constraint types (from the ID table above) it was reformulated into — e.g. an `==>`
implication with a binary indicator ends up as an indicator constraint (`_indle`) plus a
linear equality (`_lineq`) and an `or` constraint tying them together.

## Automatic solution checking

Every MP solver automatically checks the returned solution against the model's tolerances.
By default this covers the AMPL-side and solver-side models only; option `sol:chk:mode`
(see `feature-and-solver-options-guide.md`) extends checking to intermediate reformulated
expressions too, with related options to tune tolerances.

There are two checking modes, equivalent for linear and quadratic models and potentially
different for models with other nonlinear expressions:

- **"Realistic"** — trusts values the solver computed and reported via an auxiliary
  variable, up to a tolerance. This matches what most solvers themselves consider
  feasible: e.g. for `C: y<=6 or z>=10;` with `y=6.0000000001, z=9.9999999999` and a
  typical 1e-6 feasibility tolerance, the solver (and the realistic check) consider the
  disjunction satisfied — even though AMPL, asked to `display C.val` with those exact
  values substituted, reports `C.val = 0` (violated), because it recomputes from scratch
  with no tolerance awareness.
- **"Idealistic"** — recomputes every expression tree from scratch (the same way AMPL does
  for `display`) and compares to the solver's reported value. This warns about the case
  above even when the realistic check passes — the solver's solution is valid up to *its*
  tolerance, but a tolerance-unaware recomputation (e.g. later re-evaluating the model in
  plain AMPL) can disagree. By default idealistic checking only covers objective values;
  enable it for constraints too with `chk:mode`.

### Reading a tolerance-violation warning

```
------------ WARNINGS ------------
WARNING.  2 case(s) of "Tolerance violations". One of them:
  Type                         MaxAbs [Name]   MaxRel [Name]
  objective(s)                 3E+00 [TotalSum]  2E+00 [TotalSum]
* algebraic con(s)             1E+00 [C2]      1E+00 [C2]
*: Using the solver's aux variable values.
Documentation: mp.ampl.com/modeling-tools.html#automatic-solution-check.
```

Each row names a violation type (objective, algebraic constraint, or a specific expression
like `expr '_pow'`, `expr '_ifthen'`) with its worst absolute and relative violation, and
which named entity it occurred in. A `*` prefix marks a "realistic" violation (computed
using the solver's own auxiliary-variable values); unmarked rows are "idealistic." If
`MaxRel` is blank, the constraint's right-hand side is 0 (relative violation isn't
well-defined) — judge significance from the coefficients and variable values instead. Use
`_pow`/`_ifthen`/etc. names from the "Full constraint list" table above to identify which
expression type is implicated, and the Reformulation Explorer to find *which* instance of
that expression in the original model.

To recompute and inspect the underlying values directly:

```ampl
display x, y, TotalSum, C2.slack;
display {i in 1.._ncons: _con[i].slack < -1e-3} (_conname[i], _con[i].slack);
display {i in 1.._nlogcons: !_logcon[i].val} (_logconname[i]);
```

### Remedies

For **realistic** violations, the cause is almost always a numerical-accuracy issue — see
`efficiency-and-numerical-accuracy.md`. For **idealistic** warnings, the goal is making
sure AMPL can access the solver's true objective value even for a discontinuous or
tolerance-sensitive objective; the standard remedy is introducing an explicit variable for
the objective value (worked example: AMPL Colab,
`https://colab.ampl.com/#solution-check-discontinuous-objective-function`).

## The MP2NL meta-driver

MP2NL translates MP-compatible syntax and features down to any AMPL solver — including
ones that were never built against MP. This benefits MINLP solvers (Knitro, Couenne) most
directly, but even smooth-NLP solvers like Ipopt can gain from MP2NL's automatic
reformulation of complementarity constraints or convex piecewise-linear expressions that
they wouldn't otherwise accept.

Some nonlinear solvers (Knitro, Baron, Conopt) and the legacy ASL drivers (`gurobiasl`,
`cplexasl`, `xpressasl`) support `mp2nl=1` as a direct option:

```ampl
option knitro_options 'soltype=1 mp2nl=1';
solve;
```

For any other solver, invoke `mp2nl` explicitly as the solver:

```python
ampl.solve(solver='mp2nl',
           ipopt_options='max_cpu_time 15',
           mp2nl_options='solver=ipopt cvt:compl=2 cvt:compl:eps=1e-8')
```

## General troubleshooting

**Look for a more efficient model first.** Tighten bounds; rescale coefficients; check
whether a constraint or logical condition is actually needed; try a manual reformulation
(e.g. of a logical condition) even where MP would auto-reformulate it — see
`efficiency-and-numerical-accuracy.md`. Then work through the relevant options: AMPL's own
presolve (`option presolve 0;` to disable and see the unmodified model), the solver's
presolve (e.g. `option gurobi_options 'presolve=0';`), tolerances, `numfocus`/`intfocus`,
automatic tuning, and MP's own reformulation options (`acc:...`, above).

**Export the model** at the solver level (and presolved level, where supported) to see
exactly what's being solved — same mechanism as above:

```ampl
option gurobi_options 'writeprob=disj.lp writepresolved=disj_pre.lp';
option gurobi_auxfiles rc;
solve;
```

**Contacting support.** Provide a (possibly reduced) reproduction of the model, the AMPL
solver version (`<solver> -v`, e.g. `highs -v`) or solver logs (`option <solver>_options
'version outlev=1 timing=2';`). If the model can't be reduced or shared, provide the raw
NL file instead:

```ampl
option auxfiles ''; write bdisj;
```

For solver-side support specifically, use MPS or another full-precision export format
rather than a lossy one.

## See also

- `overview-and-when-to-use.md` — what MP is and which solvers are MP-based
- `efficiency-and-numerical-accuracy.md` — the formulation and bounding practices most
  troubleshooting sessions end up applying
- `feature-and-solver-options-guide.md` — `sol:chk:mode`/`chk:mode` and other solution-check
  tuning options, plus general option mechanics (`<solver>_options`, querying with `?`)
- `../core-language/11-solver-interaction-and-suffixes.md` — presolve and `solve_result`
  mechanics that aren't MP-specific
- `../modeling-patterns/complementarity-problems.md` — the complementarity constraints
  MP2NL can reformulate for solvers without native support
