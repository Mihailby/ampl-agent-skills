# XPRESS Options

Derived from: xpress-mp-options.md, xpress-asl-options.md

Source: `raw/xpress-mp-options.md` (`https://dev.ampl.com/solvers/xpress/options.html`,
MP/native interface) and `raw/xpress-asl-options.md`
(`https://dev.ampl.com/solvers/xpress/xpressasl.html`, legacy ASL interface), both
current as of 2026-08-17.

**Caveat — the MP-interface source is incomplete.** Per `raw/sources.md`, every fetch
route for the MP options page truncated at the identical point (mid-entry in
`log:xprs_outputcontrols`), so `raw/xpress-mp-options.md` only covers options
alphabetically from `acc:` through `log:xprs_maxpagelines`. It **is** complete for
`acc:`, `alg:`, `bar:`, and `cut:`-prefixed options, but is **missing everything from
`mip:` onward**: `mip:`, `nlp:`, `out:`, `pre:`, `qcp:`, `sol:`, `tech:`, `tol:`,
`warmstart`, `writeprob`/`writesolution`. For categories below that fall in the missing
range, this file cites the option name from the (complete) ASL-interface source instead
— those are the legacy `xpressasl_options` names, which may differ from, or have no
direct 1:1 equivalent to, the modern MP-interface (`xpress_options`) name. **If the exact
current MP name matters for your use case, check
`https://dev.ampl.com/solvers/xpress/options.html` directly.**

## Passing options

Two distinct AMPL drivers exist for XPRESS, each with its own option string:

```ampl
option solver xpress;                          # MP/native interface
option xpress_options 'mipgap=1e-6 outlev=1';   # space-separated name=value pairs
solve;
```
```ampl
option solver xpressasl;                        # legacy ASL interface
option xpressasl_options 'maxtime=300 outlev=1';
solve;
```

Many MP option entries list old ASL-style short names as recognized aliases (e.g.
`alg:method` also accepts `method`, `lpmethod`, `defaultalg`) — so an ASL name cited
below sometimes also works verbatim in `xpress_options`, but this isn't confirmed for
the missing `mip:`-onward categories.

## Autotuner

ASL-only in the sources captured here (not found in the captured `mip:`-onward range of
the MP file, so unconfirmed whether an MP-native equivalent exists — check the live
docs). `tunerdir` turns the tuner on and is where results are stored; `tunermethod`
picks an LP/MIP tuning strategy; `tunermaxtime` and `tunerthreads` bound its cost.

```ampl
option xpressasl_options 'tunerdir=xpress_tuner tunermethod=1 tunermaxtime=60';
```

## MIP stopping criteria (ASL names — `mip:` range missing from MP source)

| Option | Effect |
|---|---|
| `maxtime` | Time limit in seconds (negative = hard stop; positive = stop once a feasible solution exists) |
| `miprelstop` | Stop when relative gap `\|obj − bound\| / \|bound\|` falls below this (default 1e-4) |
| `mipabsstop` | Stop when absolute gap `\|obj − bound\|` falls below this (default 0) |
| `maxnode` | Max B&B nodes explored |
| `maxmipsol` | Max integer solutions to find (0 = no limit) |

```ampl
option xpressasl_options 'maxtime=300 miprelstop=0.01 maxnode=100000 maxmipsol=5';
```

The MP file's own header example uses `mipgap=1e-6`, but the entry that would define it
falls in the missing `mip:` range — likely the MP-native relative-gap equivalent of
`miprelstop`; unconfirmed here.

## Variable/branch selection (ASL names — falls in `mip:` range missing from MP)

`varselection` scores candidate integer variables at each node (pseudocost-based, 6
strategies). `sbeffort`/`sbselect`/`sbiterlimit` tune strong branching; `pseudocost` /
`pseudocost_ud` control the initial pseudocost value and how it's updated.

```ampl
option xpressasl_options 'varselection=2 sbeffort=2';
```

## Numerical issues

`alg:numericalemphasis` (MP, present source range; aliases `numericfocus`, `numfocus`)
trades solve speed for numerical stability, 0 (speed) to 3 (strong stability), default
automatic. `alg:feastol` (MP, present) is the primal feasibility tolerance (default
1e-6). Integrality tolerance (`miptol`, default 5e-6) and matrix `scaling` (bit-vector,
default 163) fall in the missing MP range — ASL names shown here.

```ampl
option xpress_options 'alg:numericalemphasis=2 alg:feastol=1e-7';
option xpressasl_options 'miptol=1e-7 scaling=163';
```

## Cuts (MP `cut:` prefix — present in source range)

`cut:strategy` sets overall cut aggressiveness: -1 automatic, 0 none, 1 conservative, 2
moderate, 3 aggressive (more cuts = fewer nodes but more time per node). `cut:freq`
sets how often cuts are regenerated during the tree search; `cut:select` /
`cut:treeselect` give bit-vector control over which cut families run at the root vs. in
the tree (e.g. Gomory, MIR, cover, clique).

```ampl
option xpress_options 'cut:strategy=3';
```

## Concurrent and algorithm selection

`alg:method` (MP, present; aliases `method`, `lpmethod`, `defaultalg`) picks the LP
algorithm: 1 automatic, 2 dual simplex, 3 primal simplex, 4 Newton barrier. General
presolve (`presolve`, on/off + redundant-bound handling) and concurrent-MIP controls
(`mipconcursolves`, `mipconcurnodes`) fall in the missing `pre:`/`mip:` MP range — ASL
names shown. `threads`/`mipthreads` set parallelism, relevant to how concurrent solves
divide up work.

```ampl
option xpress_options 'alg:method=4';
option xpressasl_options 'presolve=1 mipconcursolves=2 threads=8';
```

## Verbosity

`outlev` (ASL; falls in the missing MP `log:`/`out:` range — the truncation point in
`raw/xpress-mp-options.md` is literally mid-entry in `log:xprs_outputcontrols`, the
likely MP equivalent) sets message level: 1 all, 2 information, 3 warnings/errors only
(default), 4 errors, 5 none.

```ampl
option xpressasl_options 'outlev=1';
```

## See also

- `../mp-library/feature-and-solver-options-guide.md` — cross-solver option-handling
  mechanics (MP driver conventions, suffixes)
- `../core-language/13-solver-selection-guide.md` — choosing XPRESS vs. other solvers
- `../modeling-patterns/integer-programming.md` — MIP gap tolerances and warm starts in
  modeling context
- Exhaustive option lists: `raw/xpress-mp-options.md`, `raw/xpress-asl-options.md`,
  `https://dev.ampl.com/solvers/xpress/options.html`,
  `https://dev.ampl.com/solvers/xpress/xpressasl.html`
