# amplpy: Installation, Modules, and Environment Setup

Derived from: ampl-modules-for-python.md, amplpy-best-practices.md

Source: `ampl-modules-for-python.md`, `amplpy-best-practices.md` (Section 1).

`amplpy` is this skill's default recommendation for driving AMPL: keep the model in a
`.mod` file and do everything else — data, options, solving, results — through this API
rather than `.dat`/`.run` files or another language's API, unless the user says
otherwise. See "Default recommendations" in `../../SKILL.md`.

## Install and activate

```bash
python -m pip install amplpy --upgrade
python -m amplpy.modules install highs gurobi         # install AMPL + any solvers as Python packages
python -m amplpy.modules activate <license-uuid>       # e.g. a free https://ampl.com/ce license
python -m amplpy.modules run ampl -vvq                 # verify activation
```

```python
from amplpy import AMPL
ampl = AMPL()          # instantiate; finds installed modules/license automatically
```

If you already have a separate AMPL installation (standalone, AMPL IDE), `amplpy` locates
its license and solvers automatically — no extra configuration needed. If you specifically
want to make sure the pip-installed modules (not the other installation) are used:

```python
from amplpy import AMPL, modules
modules.load()          # load all pip-installed modules explicitly
ampl = AMPL()
```

## Modules system: AMPL and every solver as Python packages

`amplpy.modules` treats AMPL itself and each solver as an independently installable
package — this is what makes `pip install amplpy` + `amplpy.modules install <solver>`
sufficient with no separate binary download/install step.

| Command | Purpose |
|---|---|
| `python -m amplpy.modules usage` | usage instructions |
| `... install <solver1> <solver2> ...` | install one or more modules |
| `... install base <solver> ... --upgrade` | update AMPL/solver versions (not automatic) |
| `... activate <license-uuid>` | activate a license |
| `... uninstall <solver1> ...` | remove modules |
| `... installed` | list what's currently installed |
| `... available` | list everything installable |
| `... path` | PATH-appendable value to access modules directly (not via amplpy) |
| `... find <name>` | locate a specific module file (e.g. a solver executable) |
| `... requirements` | emit a `requirements.txt` pinning currently-installed module versions |
| `... run <command>` | run any command in the modules' environment (e.g. `run ampl -v`) |

Open-source: `highs`, `cbc`, `coin` (bundles CBC/Couenne/Ipopt/Bonmin), `open` (all
open-source solvers). Commercial: `baron`, `conopt`, `copt`, `cplex`, `gurobi`, `knitro`,
`lgo`, `lindoglobal`, `loqo`, `minos`, `mosek`, `snopt`, `xpress`. Also available:
`gokestrel` (NEOS Server client), `amplgsl`/`plugins` (AMPL function/table-handler
plugins).

The whole interface is also available programmatically, e.g. for scripted
provisioning:

```python
from amplpy import modules
modules.installed()          # -> ['base', 'gurobi', 'highs']
modules.available()          # -> full list of installable module names
modules.install('cplex')
modules.uninstall('cplex')
```

**Modules are not auto-updated** — re-run `install ... --upgrade` deliberately when you
want newer AMPL/solver versions; nothing does this silently on `pip install --upgrade
amplpy`.

## Google Colab / Jupyter

```python
%pip install -q amplpy
from amplpy import AMPL, ampl_notebook
ampl = ampl_notebook(
    modules=["coin", "highs", "cplex", "gurobi"],   # solvers to install
    license_uuid="default")                          # Colab's built-in free CE license
```

`ampl_notebook(...)` is a one-call convenience wrapper (install + activate + instantiate +
register notebook magics) specific to the notebook environment; outside a notebook (or if
you already have a license configured) `AMPL()` alone is equivalent to the non-notebook
flow above.

## Using AMPL solvers from other Python modeling tools

Because the `.nl` format (AMPL's solver interchange format) has been adopted beyond AMPL
itself, `amplpy.modules`-installed solvers can be pointed at from other tools, e.g. Pyomo:

```python
from amplpy import modules
import pyomo.environ as pyo
solver = pyo.SolverFactory("highsnl", executable=modules.find("highs"), solve_io="nl")
```

Worth knowing if a project mixes AMPL and Pyomo models, but note the caveat that Pyomo
(being pure Python, without AMPL's C-based instantiation and presolve) is typically
substantially slower end-to-end than solving the equivalent model natively through AMPL —
prefer the native AMPL path when performance matters. Pyomo itself is an independent
project; support for Pyomo-specific issues unrelated to a particular solver driver should
go through Pyomo's own channels, not AMPL's.

## See also

- Everyday amplpy workflow once installed (loading models, loading data, solving,
  retrieving results): `core-workflow-best-practices.md`
- Full `AMPL`/`Entity`/`Variable`/etc. class method reference: `api-reference-classes.md`
