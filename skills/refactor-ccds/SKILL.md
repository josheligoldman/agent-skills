---
name: refactor-ccds
description: Reorganize an existing, non-ccds repository into the cookiecutter-data-science (ccds) structure, on a new branch, preserving history via git mv. Resolves ambiguous file mappings through a grilling session before moving anything. Use when the user wants to migrate/refactor an existing data science repo into ccds style, when a repo is "a mess" and needs structure, or when the new-ds-project skill refused because the target directory wasn't empty.
---

# Refactor an existing repo into ccds structure

Reorganizes a repo's file layout to match cookiecutter-data-science
(https://cookiecutter-data-science.drivendata.org/) in place, without losing
git history. See
[docs/adr/0002-refactor-safety.md](../../docs/adr/0002-refactor-safety.md)
for why this is branch + `git mv` + grill-before-moving, rather than guessing
or asking ad hoc mid-refactor.

## Step 1 — Require a clean working tree

Run `git status`. If there are uncommitted changes, ask the user to commit or
stash them first — don't proceed on top of unrelated in-progress work.

## Step 2 — Create a new branch

```bash
git checkout -b refactor/ccds-structure
```

If that branch name already exists, append a suffix (`-2`, etc.) rather than
overwriting it.

## Step 3 — Inventory the repo

List the top-level files and directories, and enough of their contents to
classify each one: data files, notebooks, scripts/modules, model artifacts,
generated reports/figures, docs, config/metadata files.

## Step 4 — Propose a mapping using standard ccds heuristics

| Current content | ccds destination |
|---|---|
| Loose data files (`.csv`, `.parquet`, raw dumps) | `data/raw/` (mark immutable going forward) |
| Third-party/downloaded datasets | `data/external/` |
| Intermediate transformed data | `data/interim/` |
| Model-ready data | `data/processed/` |
| `.ipynb` notebooks | `notebooks/`, renamed to `<n>.<initials>-<slug>.ipynb` if not already |
| Reusable Python modules/scripts | `src/<module_name>/{data,features,modeling,visualization}/`, split by what they actually do |
| Trained model artifacts (`.pkl`, `.joblib`, `.pt`, `.h5`, ...) | `models/` |
| Output charts/images | `reports/figures/` |
| Written analyses/write-ups | `reports/` |
| Data dictionaries, manuals | `references/` |
| Docs source | `docs/` |
| `README`/`LICENSE`/dependency files | Merge into ccds's root files — never overwrite existing content, append/merge instead |

## Step 5 — Isolate ambiguous mappings

Anything that doesn't fit a heuristic cleanly — a script's name doesn't
indicate its role, a module mixes feature engineering with modeling, a
notebook has no clear topic — goes on a separate list. Do not guess.

## Step 6 — Grill the ambiguous set before moving anything

If the ambiguous list is non-empty, run a session following the `grilling`
skill's own method: map the ambiguous items as a design tree, ask the whole
current frontier as one round of numbered questions with your recommended
answer for each, wait for the user's answers, and repeat until every
ambiguous item has a settled destination. Do not move a single file until
this is done.

## Step 7 — Confirm the final plan

Once there's no ambiguity left, show the user the complete old-path →
new-path table and wait for explicit confirmation before touching the
filesystem — unless the user's original request already said to skip
confirmation for this run.

## Step 8 — Execute

- Use `git mv` for every tracked file so history follows it. Fall back to
  `mv` + `git add` only for untracked files.
- Create any missing standard ccds directories (`data/{raw,interim,processed,external}`,
  `notebooks/`, `src/<module_name>/`, `models/`, `reports/figures/`,
  `references/`, `docs/`), with `.gitkeep` where a directory would otherwise
  be empty.
- Create missing root files (`LICENSE`, `Makefile`, `pyproject.toml`,
  `README.md` skeleton) only if they don't already exist; merge/append into
  existing ones instead of overwriting.

## Step 9 — Fix what the move broke

Grep for imports/paths referencing anything that moved, and update them so
the repo still runs.

## Step 10 — Commit and report

Commit the reorganization on the branch with a clear message. Report a
summary of what moved where, and a follow-up TODO list for anything flagged
during Step 5 that needs more manual judgment than a mapping decision (e.g.
"`analysis.py` mixes three concerns — consider splitting it further").

Tell the user the branch name and that it hasn't been merged or pushed —
that decision is theirs.
