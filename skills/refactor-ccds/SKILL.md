---
name: refactor-ccds
description: Reorganize an existing, non-ccds repository into the cookiecutter-data-science (ccds) structure, on a new branch, preserving history via git mv. Resolves ambiguous file mappings through a grilling session before moving anything, gated by the existing test suite. Use when the user wants to migrate/refactor an existing data science repo into ccds style, when a repo is "a mess" and needs structure, or when the new-ds-project skill refused because the target directory wasn't empty.
---

# Refactor an existing repo into ccds structure

Reorganizes a repo's file layout to match cookiecutter-data-science
(https://cookiecutter-data-science.drivendata.org/) in place, without losing
git history. See
[docs/adr/0002-refactor-safety.md](../../docs/adr/0002-refactor-safety.md)
for why this is branch + `git mv` + grill-before-moving, rather than guessing
or asking ad hoc mid-refactor.

## Core rule: structural moves only, never a silent behavior change

This skill moves code and reorganizes files. It does not rewrite a repo's
working internals, swap out its frameworks, or retrofit its existing code
onto house patterns (like the `Protocol`-based swappable-implementations
convention) — those are separate, later efforts, not something to attempt
mid-move. Default to zero behavior change. If completing a structural move
genuinely requires a behavior change (not just a changed import path), stop
and ask the user as an explicit question — never decide unilaterally, and
never silently skip the move to avoid asking. This rule is why Steps 7-10
below exist as separate checkpoints rather than being folded into one bulk
"execute" step.

The one deliberate exception is Step 10 (tooling): migrating a repo's
existing tooling onto a house convention (e.g. an existing config system
onto Tyro) is a real behavior change, but it's *always* offered rather than
silently skipped — see Step 10 for how that's reconciled with this rule.

## Step 1 — Require a clean working tree

Run `git status`. If there are uncommitted changes, ask the user to commit or
stash them first — don't proceed on top of unrelated in-progress work.

## Step 2 — Create a new branch

```bash
git checkout -b refactor/ccds-structure
```

If that branch name already exists, append a suffix (`-2`, etc.) rather than
overwriting it.

## Step 3 — Baseline the existing test suite

If the repo has tests, run them now, before anything else changes, and
record the result as-is — including if it's already red. This is the
reference point Step 9 checks the post-move state against; without it, a
pre-existing failure could be mistaken for something the refactor broke, or
vice versa. If there's no test suite, note that explicitly — Step 9's
regression check doesn't apply, and that's worth flagging in the Step 11
report as a gap, not silently ignoring.

## Step 4 — Inventory the repo

List the top-level files and directories, and enough of their contents to
classify each one: data files, notebooks, scripts/modules, model artifacts,
generated reports/figures, docs, config/metadata files.

While inventorying, also classify the repo's overall shape:

- **Simple**: loose scripts/notebooks/data files with no real internal
  package structure. The standard mapping table in Step 5 handles this
  directly.
- **Complex**: an existing, differently-organized package that already has
  its own interfaces, shared components across multiple implementations,
  or framework choices (a config system, an ML framework, a CLI) that
  differ from house convention. Treat the Step 5 table as a starting point,
  not a complete answer — expect more items to land on Step 6's ambiguous
  list, and read the "Core rule" above twice before Step 8.

Flag any **dead/orphaned code** found during inventory — directories or
files with no live source (e.g. a directory containing only compiled
artifacts like `.pyc` files, or files nothing imports) — as their own
category, separate from things that need moving. Never delete these
automatically; they go into the Step 7 confirmation as an explicit
"proposed for deletion" list the user signs off on, same as any other part
of the plan.

## Step 5 — Propose a mapping using standard ccds heuristics

| Current content | ccds destination |
|---|---|
| Loose data files (`.csv`, `.parquet`, raw dumps) | `data/raw/` (mark immutable going forward) |
| Third-party/downloaded datasets | `data/external/` |
| Intermediate transformed data | `data/interim/` |
| Model-ready data | `data/processed/` |
| `.ipynb` notebooks | `notebooks/`, renamed to `<n>.<initials>-<slug>.ipynb` if not already |
| Reusable Python modules/scripts | `src/<module_name>/{data,modeling,visualization}/` — one subpackage per pipeline stage (see [CLAUDE-CCDS.md](../../CLAUDE-CCDS.md)'s `src/<module_name>/` layout section for the canonical file split within each: `data/make_dataset.py`+`validators.py`, `modeling/base.py`+`train.py`+`predict.py`+`architectures.py`, `visualization/plots.py`). There is deliberately no `features/` — every project here is representation learning, so feature/embedding code is a `modeling/` component instead. |
| Low-level code shared by multiple implementations within one stage (shared layers, utilities used by more than one `Backbone`/`Model`/`Embedder`) | `<stage>/common/` (e.g. `modeling/common/`) — see `CLAUDE-CCDS.md`'s "`common/` for genuinely shared code" note. Don't force-split this across stages just because it's reusable. |
| Trained model artifacts (`.pkl`, `.joblib`, `.pt`, `.h5`, ...) | `models/` |
| Output charts/images | `reports/figures/` |
| Written analyses/write-ups | `reports/` |
| Data dictionaries, manuals | `references/` |
| Existing `docs/adr/`, `docs/specs/`, `CONTEXT.md`, or similar documentation | Left exactly where it is, untouched — these describe past decisions and stay as historical record. New entries get added going forward for decisions *this refactor* makes; nothing existing gets edited or reorganized as a side effect of the file-layout move. |
| Other docs source | `docs/` |
| `README`/`LICENSE`/dependency files | Merge into ccds's root files — never overwrite existing content, append/merge instead |

## Step 6 — Isolate ambiguous mappings

Anything that doesn't fit a heuristic cleanly — a script's name doesn't
indicate its role, a module mixes feature engineering with modeling, a
notebook has no clear topic — goes on a separate list. Do not guess.

## Step 7 — Grill the ambiguous set before moving anything

If the ambiguous list is non-empty, run a session following the `grilling`
skill's own method: map the ambiguous items as a design tree, ask the whole
current frontier as one round of numbered questions with your recommended
answer for each, wait for the user's answers, and repeat until every
ambiguous item has a settled destination. Do not move a single file until
this is done.

## Step 8 — Confirm the final plan

Once there's no ambiguity left, show the user the complete plan and wait
for explicit confirmation before touching the filesystem — unless the
user's original request already said to skip confirmation for this run.
The plan includes:
- The old-path → new-path table.
- The dead-code deletion list from Step 4, called out separately and
  clearly as deletions, not moves.
- Any point where completing a move would require an actual behavior
  change (not just an import path change) — flagged per the Core rule
  above, with your recommendation, so the user is deciding it here rather
  than discovering it after the fact.

## Step 9 — Execute

- Use `git mv` for every tracked file so history follows it. Fall back to
  `mv` + `git add` only for untracked files.
- Delete only what was explicitly confirmed in Step 8's dead-code list.
- Create any missing standard ccds directories (`data/{raw,interim,processed,external}`,
  `notebooks/`, `src/<module_name>/{data,modeling,visualization}/`,
  `models/`, `reports/figures/`, `references/`, `docs/`), with `.gitkeep`
  where a directory would otherwise be empty.
- Create missing root files (`LICENSE`, `Makefile`, `pyproject.toml`,
  `README.md` skeleton) only if they don't already exist; merge/append into
  existing ones instead of overwriting.

## Step 10 — Fix what the move broke, then re-check against baseline

Grep for imports/paths referencing anything that moved, and update them so
the repo still runs. Then, if Step 3 found a test suite, run it again and
diff the result against the Step 3 baseline:
- Anything that was passing and is now failing is a regression the move
  introduced — this blocks Step 12 (commit) until fixed. A structural move
  should not change test outcomes.
- Anything that was already failing at baseline and still is isn't this
  skill's problem to fix — note it in the Step 12 report so it isn't
  confused with something the refactor caused.
- If there was no baseline (no test suite), skip this comparison — already
  flagged as a gap in Step 3.

Also restructure `tests/` to mirror wherever `src/<module_name>/` ended up
(`tests/data/`, `tests/modeling/`, `tests/visualization/`, each with
`__init__.py`), moving existing tests into the subpackage they now belong
to, per `CLAUDE-CCDS.md`'s Testing section.

## Step 11 — Offer to add or migrate house tooling

Ask the user whether to also bring the repo up to this project's tooling
conventions (see [CLAUDE-CCDS.md](../../CLAUDE-CCDS.md) for the rationale
behind each). Two distinct cases, both always asked about — this is the
Core rule's one deliberate exception, since tooling migration is a real
behavior change but withholding the option isn't the safer choice here:

- **Missing entirely** — add it, same as before:
  - `pre-commit` (`ruff` + `mypy` + `nbstripout`), installed and run once
    (`pre-commit install`, `pre-commit run --all-files`) so existing files
    are brought into compliance rather than failing on the first future
    commit.
  - `.github/workflows/ci.yml` (tests + lint + type-check + `pip-audit`) if
    the repo doesn't already have CI.
  - `pandera` stub schemas in `data/validators.py` if that file didn't
    already exist.
  - `CITATION.cff` and `docs/adr/` (with the same short explainer used by
    `new-ds-project`) if missing.
  - A `set_seed()` helper in `config.py` (or wherever paths/env are already
    centralized) if one doesn't already exist.
  - `modeling/base.py` (`Model` `Protocol` and `Embedder`/`Encoder`
    `Protocol`) and `modeling/architectures.py` (Tyro config dataclasses) if
    the existing modeling code doesn't already follow the
    swappable-implementations pattern.
  - `hypothesis` and `pytest-cov` as dev/test dependencies, and `"C901"`
    (McCabe complexity) added to `[tool.ruff.lint]`'s `select` list, if not
    already present — see `CLAUDE-CCDS.md`'s Testing and Code Quality
    Tooling sections.
  - `.PHONY` declarations on the Makefile's targets, and
    `scripts/submit_train.sbatch` (per `CLAUDE-CCDS.md`'s Cluster
    submission section) if the repo runs training on a SLURM cluster and
    doesn't already have a checked-in submission script.
  - If the repo already uses HF Hub for its dataset but `data/make_dataset.py`
    doesn't push the regenerated result there by default, offer to wire in
    the `push: bool = True` default per `CLAUDE-CCDS.md`'s "Publishing the
    regenerated dataset to HuggingFace Hub" section. Don't newly introduce
    HF Hub to a repo that doesn't already use it for data — that's a data
    -infrastructure choice for the user to make separately, not something
    this refactor should propose unprompted.
- **Already present, but conflicting** — a working config system where
  house convention says Tyro, a different logger/CLI/experiment tracker,
  a `[tool.mypy]` section with a global `ignore_missing_imports = true`
  instead of the house per-module-override policy, etc. Always ask, and
  recommend migrating to the house convention as the default answer — but
  if the user declines, keep what's there. Either way, record the outcome
  in the Step 12 report so a "not yet migrated" gap is a visible, tracked
  decision rather than something nobody notices later.

For anything declined (in either case), note it in the Step 12 report as a
follow-up rather than silently skipping it.

## Step 12 — Commit and report

Commit the reorganization (and any tooling added or migrated in Step 11) on
the branch with a clear message. Report:
- A summary of what moved where, and what was deleted (from Step 9's
  confirmed dead-code list).
- The test outcome from Step 10 — regressions fixed, pre-existing failures
  noted separately, or "no test suite existed" if that was the case.
- What tooling was added, migrated, or declined (from Step 11), including
  any conflicting-tooling gaps left in place by the user's choice.
- A follow-up TODO list for anything flagged during Step 6 that needs more
  manual judgment than a mapping decision (e.g. "`analysis.py` mixes three
  concerns — consider splitting it further"), and anything explicitly
  scoped out per the Core rule (framework migrations, `Protocol`-conformance
  rewrites of existing model code) as separate, later work.

Tell the user the branch name and that it hasn't been merged or pushed —
that decision is theirs.
