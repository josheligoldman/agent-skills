# Agent Skills — ccds tooling

Skills for scaffolding new projects and refactoring existing ones into the
cookiecutter-data-science (ccds) structure, plus the conventions that keep
an agent honoring that structure during everyday work afterward.

## Language

**ccds project**:
A repository whose layout matches the cookiecutter-data-science template:
`data/{raw,interim,processed,external}`, `notebooks/`, `src/<module_name>/`,
`models/`, `reports/figures/`, `references/`, `docs/`.
_Avoid_: data science repo, ML project structure

**Scaffold**:
Generate a brand-new ccds project from nothing, via the `ccds` CLI. Only
valid when the target directory is empty.
_Avoid_: create, refactor, migrate, initialize

**Refactor** (this context):
Reorganize an existing, non-ccds repo's files into the ccds structure in
place, preserving git history via `git mv`, on a new branch.
_Avoid_: migrate, scaffold, convert, restructure

**Ambiguous mapping**:
A file or piece of code encountered during a refactor whose correct
destination in the ccds structure isn't obvious from its name or location
alone. Resolved via a grilling session with the user before anything is
moved.
_Avoid_: unclear file, edge case

**Live schema check**:
Re-fetching ccds's own current config schema (`ccds.json`) from upstream at
scaffold time, instead of trusting a hardcoded copy, so `new-ds-project`
can't silently drift from what the `ccds` CLI actually supports.
_Avoid_: version check, compatibility check
