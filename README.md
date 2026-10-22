# AMPL Knowledge Skill

A portable Agent Skill that gives an LLM specialized knowledge of AMPL — syntax, the amplpy/ampls-api Python interfaces, modeling best practices, and tutorial content — to help OR and optimization practitioners (beginners through advanced) write better AMPL models.

Agent Skills are an open, filesystem-based format (a folder with a `SKILL.md` file plus reference material) for extending LLM agents with specialized knowledge. This skill has no dependency on any particular vendor or platform — it's plain Markdown reference content intended to work with any LLM tool or agent runtime that supports the Agent Skills format.

This repository is the distribution point for the skill. It contains the finished, ready-to-install package — not the source material or build process used to create it.

## What's in this repo

```
skill/
  SKILL.md          # Entry point — routes questions to the right reference file
  references/
    amplpy-api/              # amplpy Python API: AMPL, Variable, Set, Constraint, ...
    ampls-solver-interfaces/ # ampls-api: solver interfaces, callbacks, CPLEX/Gurobi
    tutorials/                # Basic commands, modeling, scripting, data
    best-practices/           # Style guide, FAQs
    book/                     # AMPL textbook content, split by chapter/topic
    examples/                 # Reference models (e.g. Netlib)
CHANGELOG.md
```

## Installing the skill

The `skill/` folder is the installable unit. How you load it depends on your agent or tool's Agent Skills support — generally either:
- **Filesystem-based**: copy the contents of `skill/` into wherever your tool looks for installed skills (a local skills directory).
- **Upload-based**: zip the `skill/` folder and upload it wherever your tool accepts skill packages.

Consult your specific tool's documentation for the exact mechanism it expects.

## Versioning

Releases are tagged (`v1.0.0`, `v1.1.0`, ...) at each point a new version is distributed. See [CHANGELOG.md](./skill/CHANGELOG.md) for what changed in each version. There's no auto-update mechanism — updating an already-installed copy of the skill means re-downloading and re-uploading/replacing it wherever it's installed.

## Scope and limits

This skill is built from AMPL documentation, tutorials, and reference material current as of the date of each release — it reflects a snapshot, not a live feed. It does not execute or solve AMPL models; it provides knowledge to help write, structure, and troubleshoot them correctly.

