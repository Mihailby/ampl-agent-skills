# FAQs: Licensing, Installation, and Deployment

Derived from: ampl-faqs.md

Source: `ampl-faqs.md` (AMPL Optimization's official FAQ page, dev.ampl.com/more/faq.html).

**Scope note**: this file covers licensing/installation/deployment questions, which is
what the underlying FAQ page actually addresses — it is not a modeling-troubleshooting
guide. For debugging actual model behavior (infeasibility, unexpected solver results,
nonlinear-solve failures, singularities), see the "pitfalls"/gotcha material embedded
throughout `core-language/` and `modeling-patterns/` — most notably
`modeling-patterns/nonlinear-programs.md` (function range violations, multiple local
optima), `core-language/11-solver-interaction-and-suffixes.md` (presolve-detected
infeasibility, solve_result codes), and `core-language/09-display-and-output-commands.md`
(the solver-noise-near-zero gotcha). No dedicated raw source in this project's corpus
covers general model-debugging FAQs beyond what those files already contain.

## What AMPL is (quick framing)

AMPL is an algebraic modeling language for large-scale optimization (production,
distribution, blending, scheduling, and similar problem classes), with an interactive
command environment for formulating models, connecting to a wide range of open-source and
commercial solvers, and examining solutions. Solver-switching is a design goal — the same
model works against different solvers with minimal change (`option solver ...;` — see
`core-language/08-commands-modifying-models-and-data.md`). APIs exist for several
languages (Python, R, and others — `amplpy/`), letting one model be developed once and
driven from whichever language is convenient, including deploying to a different language
later without changing the model itself.

**Free access options**: an Academic License (ampl.com/academia) for students,
researchers, and instructors — full AMPL plus commercial solvers, no cost. A Community
Edition License (ampl.com/ce) for personal or commercial-prototyping use with open-source
solvers via the AMPL APIs; 30-day trials exist for individual commercial solvers.
Installation starts with an account at portal.ampl.com, then the installation guide at
dev.ampl.com/ampl/install.html.

## Cloud licenses: how the lease mechanism works

AMPL's cloud licensing uses short-term leases (5 minutes each, auto-renewed) rather than
static long-term license files — deliberately, to support dynamic environments like Docker
containers where the "machine" a license is tied to may be created and destroyed
frequently. Mechanism: the current `ampl.lic` file plus a machine fingerprint are sent to
a REST endpoint (hosted across multiple cloud providers); a successful renewal replaces
`ampl.lic` with a fresh lease. This works identically on native Windows/macOS machines and
inside containers — the only hard requirement is network connectivity to the licensing
endpoint.

**What's collected in the fingerprint**: CPU core count, CPU thread count available to the
process (relevant since a container may be restricted to a subset of the host's threads).
From this, the licensing system can track, at any moment: total concurrent active leases,
total CPU threads across those leases, total distinct underlying machines, total CPU cores
across those machines.

**Exceeding license limits**: no automatic enforcement — renewal requests are logged, and
if a license appears to exceed its limits for a sustained period, the account holder is
contacted to discuss upgrading. Brief, transient over-limit periods (e.g. containers
rapidly scaling up/down) are expected and not flagged as a problem. Check current cloud
service status at status.ampl.com.

## Docker deployment

```dockerfile
FROM python:3.9-slim-bullseye
RUN python -m pip install amplpy --no-cache-dir
RUN python -m amplpy.modules install highs gurobi --no-cache-dir
# Only needed for non-root containers:
ENV AMPLKEY_RUNTIME_DIR /tmp/ampl/
ENV AMPL_LICFILE /tmp/ampl/ampl.lic
```

No official AMPL base image is provided — deliberately, so any base image can be used;
the example above is Python-specific, but the same `amplpy.modules install` pattern
(`amplpy/getting-started-and-installation.md`) works from any base image with Python
available. The `AMPLKEY_RUNTIME_DIR`/`AMPL_LICFILE` environment variables matter
specifically for containers that don't run as root, where the default license-file
location may not be writable.

## Amplbot

An AI assistant (accessible free via a ChatGPT GPT) specifically aimed at the AMPL
modeling workflow: learning AMPL, formulating models from natural-language problem
descriptions, and generating ready-to-run `amplpy` code. Worth mentioning to a user
looking for interactive/conversational help alongside this skill's static reference
material — the two are complementary, not competing (this skill's reference content is
static and precise; Amplbot is generative and can draft from a natural-language
description).

## Adding the AMPL installation directory to PATH

Needed when AMPL was installed to a non-standard location and isn't being found
automatically.

**From Python** (works cross-platform, including Windows where the variable name is
`Path` not `PATH`):
```python
import os
os.environ["PATH"] += os.pathsep + "/complete/path/to/ampl/installation/directory/"
```

**Linux** (add to `~/.profile` or `~/.bashrc`):
```bash
export PATH="/complete/path/to/ampl.linux-intel64/":$PATH
```

**macOS** (add to `~/.zshrc`):
```bash
export PATH="/complete/path/to/ampl.macos64/":$PATH
```

**Windows**: add the install directory (e.g. `C:\ampl.mswin64\`) to the `Path`
environment variable via System Properties — see
https://www.computerhope.com/issues/ch000549.htm for the click-through steps.

## Where to go for licensing/installation problems this file doesn't cover

The live FAQ page links out to a deeper licensing-support tree, not reproduced in full
here (titles/URLs only, since this is a pointer index, not explanatory content in its own
right):

- Dynamic License Troubleshooting (general licensing notes; common issues like license
  renewal errors, "Bundle Expired" messages, demo license persisting after full
  activation, "No AMPL License for This Machine" in synced folders like OneDrive;
  extending solver trials; contact support)
- Fingerprint Instructions for Static Licenses (Windows/Linux/macOS)
- Legacy Floating License Manager (installation, manager commands, configuration
  directives, troubleshooting) — relevant only for older, non-cloud floating-license
  deployments, not the lease-based cloud mechanism described above

If a licensing/installation issue isn't covered by this file's summary above, treat these
titles as a map of where the answer likely lives on dev.ampl.com/help/, rather than
guessing — this project's raw corpus doesn't include the full content of those
sub-pages.
