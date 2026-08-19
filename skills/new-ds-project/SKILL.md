---
name: new-ds-project
description: Scaffold a brand-new data science / ML project using the official cookiecutter-data-science (ccds) template as a base, then layer this project's house conventions on top (subpackage-per-stage layout, Tyro, PyTorch Lightning, wandb, pandera, pre-commit, CI). Config options are fetched live from upstream every run. Use when the user asks to start a new data science, ML, or research project, wants a repo scaffolded "cookiecutter data science" style, or mentions ccds/cookiecutter-data-science. Refuses on a non-empty target directory — see the refactor-ccds skill for existing repos.
---

# New Data Science Project (ccds + house conventions)

Scaffolds a new project using the real `ccds` CLI (the maintained successor
to plain `cookiecutter` for this template — see
https://cookiecutter-data-science.drivendata.org/) as a starting point, then
applies this repo's own conventions on top of the bare upstream scaffold.
See [CLAUDE-CCDS.md](../../CLAUDE-CCDS.md) for the full rationale behind
every convention referenced below (subpackage layout, `tyro`, PyTorch
Lightning, wandb, pandera, pre-commit, CI, etc.) — this skill file
describes *how to apply* those conventions during scaffolding, not why
they were chosen.

Also see [docs/adr/0001-live-schema-sync.md](../../docs/adr/0001-live-schema-sync.md)
for why the ccds config schema is re-fetched live every run instead of
hardcoded.

## Step 1 — Check the target directory first

Before anything else, check whether the target directory exists and is
non-empty. **If it has content, refuse and point the user to the
`refactor-ccds` skill instead** — this skill is for brand-new projects only,
never for reorganizing something that already exists.

## Step 2 — Ensure `ccds` is installed

```bash
ccds --version
```

If missing, tell the user you're installing it, then:

```bash
pipx install cookiecutter-data-science
```

If `pipx` itself isn't available, install it first (`brew install pipx` on
macOS, then `pipx ensurepath`), then retry.

## Step 3 — Fetch the live config schema

Fetch the current schema before asking the user anything, so the choices you
present always match what `ccds` actually supports today:

```
https://raw.githubusercontent.com/drivendataorg/cookiecutter-data-science/master/ccds.json
```

Parse it as JSON. For each field:
- If the value is a list (e.g. `environment_manager`, `dependency_file`,
  `open_source_license`, `docs`, `include_code_scaffold`), the **first
  element is upstream's default** — use it unless a house requirement
  overrides it (see below).
- `testing_framework` and `linting_and_formatting` are house requirements,
  not upstream defaults — see Step 4.
- If the value is a plain string that reads as a real default (e.g.
  `python_version_number: "3.10"`), use it as the default.
- If the value is a plain string that's obviously a placeholder (e.g.
  `project_name: "project_name"`, `description: "A short description..."`,
  `author_name: "Your name..."`), it must be asked from the user, never used
  literally.
- `dataset_storage` is a list of single-key objects, some with nested
  sub-fields (`s3` wants `bucket`/`aws_profile`, `azure` wants `container`,
  `gcs` wants `bucket`). Default to `none` unless the user mentions cloud
  data storage.

If the fetch fails (offline, upstream down, or the file no longer parses the
way this skill expects), tell the user and fall back to running `ccds`
fully interactively instead of guessing at a schema you can't verify.

## Step 4 — Gather project parameters

Ask the user only for what you can't reasonably default:
- **project_name**, **description** — always ask.
- **author_name** — try `git config user.name` first, else ask.
- Target parent directory.
- Whether the project stores its dataset on HuggingFace Hub — ask, default
  no. If yes, get the target `repo_id` for the processed dataset (this
  decides Step 6's `data/make_dataset.py` push wiring, Step 7's
  dependencies, and Step 12's `.env.example` entry).

Force these two regardless of upstream's listed default order — they're
required by this project's house conventions (`CLAUDE-CCDS.md`'s code
quality tooling section), not a matter of preference per project:
- `testing_framework = pytest`
- `linting_and_formatting = ruff`

For everything else, use the live-fetched defaults and tell the user what you
picked so they can override (e.g. "defaulting to `pyproject.toml` +
`ruff` + `pytest` — say if you want something else").

If the user asks for `uv` or `poetry` workflows, set `environment_manager`
accordingly and prefer `pyproject.toml` for `dependency_file`.

## Step 5 — Run it non-interactively

Pass values as key=value extra context so nothing hangs on stdin:

```bash
ccds --no-input -o <parent_dir> \
  project_name="<name>" \
  description="<description>" \
  author_name="<author>" \
  python_version_number="<version>" \
  environment_manager=<manager> \
  dependency_file=<file> \
  testing_framework=pytest \
  linting_and_formatting=ruff \
  open_source_license="<license>" \
  docs=<docs> \
  include_code_scaffold=Yes \
  dataset_storage=none
```

If a field name from the live schema doesn't match what's shown above (the
template evolved further since this skill was written), adapt the command to
the field names you actually fetched — don't silently drop a field just
because it's unfamiliar.

If `--no-input` errors, fall back to running `ccds -o <parent_dir>`
interactively and answer each prompt on the user's behalf using the gathered
values, confirming with the user as you go.

## Step 6 — Restructure `src/<module_name>/` into subpackages

Upstream ccds generates flat files (`config.py`, `dataset.py`, `features.py`,
`modeling/train.py`, `modeling/predict.py`, `plots.py`). Reorganize these
into this project's subpackage-per-stage convention:

- `dataset.py` → `data/make_dataset.py`; add `data/__init__.py` and a stub
  `data/validators.py` (an empty `pandera` `DataFrameSchema` the user fills
  in per dataset). If the project uses HF Hub (Step 4), wire in a
  `push: bool = True` param that pushes the resulting dataset to the given
  `repo_id` by default when the script finishes — per `CLAUDE-CCDS.md`'s
  "Publishing the regenerated dataset to HuggingFace Hub" section. Nothing
  else in `src/<module_name>/` ever imports `datasets` or pulls from HF Hub.
- Delete `features.py` — this convention never scaffolds a `features/`
  subpackage; every project here is representation learning, so
  feature/embedding computation is always a `modeling/` component instead
  (see `CLAUDE-CCDS.md`).
- `modeling/` is already a subpackage. Add a `modeling/base.py` stub
  defining a `Model` `Protocol` (forward-pass only — see `CLAUDE-CCDS.md`'s
  "Swappable implementations" section) and an `Embedder`/`Encoder`
  `Protocol` — and a stub `modeling/architectures.py` holding at least two
  concrete `nn.Module` implementations paired with their own Tyro config
  dataclasses, unioned into one `ModelConfig`. Add `modeling/module.py`
  (a `LightningModule` subclass wrapping a `Model` with loss/optimizer/
  metrics) and `modeling/data_module.py` (a `LightningDataModule` stub for
  batching/splits) — see `CLAUDE-CCDS.md`'s "Training loop" section for the
  full pattern. **Rewrite `train.py`** to drop its `typer` stub and instead
  build `model = config.build()` from `tyro.cli(ModelConfig)`, wrap it in
  the project's `LightningModule`, build the `LightningDataModule`, and run
  `Trainer.fit(...)` with a `WandbLogger(log_model="all")` and a
  `ModelCheckpoint` callback writing to `models/` — the Model/architectures
  pattern is a standing default from project inception, so `train.py` is a
  config-heavy stage from day one, not something to leave as ccds's plain
  single-path-args stub. Give `ModelConfig`'s union a default variant so
  bare `make train` still runs without requiring a subcommand choice.
  `predict.py` stays a plain `tyro.cli(main)` stub (loading a checkpoint
  path, an input-data path, and an output path, then calling
  `LightningModule.load_from_checkpoint(...)`) unless it independently
  crosses the config-heavy threshold later.
- `plots.py` → `visualization/plots.py`; add `visualization/__init__.py`.
  No `base.py` here by default — `visualization/` and `data/` only get a
  `Protocol` interface once a project actually needs to swap
  renderers/sources, per `CLAUDE-CCDS.md`.
- Fix any imports that referenced the old flat paths (e.g.
  `<module_name>.dataset` → `<module_name>.data.make_dataset`), including in
  the Makefile (Step 14) and README.
- Seeding uses Lightning's own `seed_everything(seed)` — no hand-rolled
  `set_seed()` helper to add to `config.py`. Call it at the top of
  `modeling/train.py`'s `main()`, before building the `Model`.
- **Mirror the new layout under `tests/`.** Upstream `ccds` generates
  `tests/` matching its old flat `src/` layout — restructure it the same
  way `src/<module_name>/` was restructured above (`tests/data/`,
  `tests/modeling/`, `tests/visualization/`,
  each with `__init__.py`), moving/renaming any ccds-generated test stub
  into the subpackage it now belongs to, per `CLAUDE-CCDS.md`'s Testing
  section. Add an empty `tests/_helpers.py` stub (docstring only, pointing
  at `CLAUDE-CCDS.md`'s Testing section) as the designated home for shared
  fixture builders once there's more than one test needing the same setup —
  don't invent fixtures speculatively before a project actually has
  repeated setup to share.

## Step 7 — Add house dependencies

Add these to the chosen `dependency_file` on top of whatever `ccds` already
added for `testing_framework`/`linting_and_formatting`: `tyro`, `lightning`,
`wandb`, `pandera`, `mypy`, `pre-commit`, `nbstripout`. Add `pip-audit`,
`hypothesis`, and `pytest-cov` as dev/test-only dependencies (not needed at
runtime). If the project uses HF Hub (Step 4), also add `datasets` and
`huggingface_hub`; otherwise leave both out entirely. `ccds`'s own scaffold
adds `typer` by default (its generated stubs import it) — remove it once
Step 6's rewrite replaces those stubs with `tyro`.

## Step 8 — Configure `ruff` and `mypy`

In `pyproject.toml`:
- Extend `[tool.ruff.lint]`'s `select` to include `"D"` (pydocstyle) and
  `"C901"` (McCabe complexity), and add:
  ```toml
  [tool.ruff.lint.pydocstyle]
  convention = "numpy"

  [tool.ruff.lint.mccabe]
  max-complexity = 10
  ```
- Add a `[tool.mypy]` section per `CLAUDE-CCDS.md`'s mypy policy: strict on
  `<module_name>.*` (`disallow_untyped_defs`, `disallow_incomplete_defs`,
  `check_untyped_defs`, `no_implicit_optional`, `warn_return_any`,
  `warn_redundant_casts`, `warn_unused_ignores` all on), plus a
  `[[tool.mypy.overrides]]` block with `ignore_missing_imports = true`
  scoped to whichever specific third-party libraries the project actually
  uses that lack type stubs — never a global `ignore_missing_imports`.

## Step 9 — Pre-commit config

Write `.pre-commit-config.yaml` with hooks for `ruff` (lint + format),
`mypy`, and `nbstripout`. After `git init` (Step 15), run
`pre-commit install` so these actually gate local commits — per house
convention this is not CI-only.

## Step 10 — GitHub Actions CI

Write `.github/workflows/ci.yml` running, on push and PR: checkout, set up
the project's Python version, install dependencies, `make lint`, `mypy .`,
`make test` (with `--cov=src/<module_name> --cov-report=term-missing` —
report-only, no `--cov-fail-under`, just visibility into what's covered),
and `pip-audit`.

## Step 11 — Data acquisition docs and `.gitignore`

Confirm `data/**` (aside from `.gitkeep` placeholders) is gitignored —
adjust `.gitignore` if the generated one doesn't already cover this. Add a
`data/README.md` with a placeholder section for the user to fill in
describing how to (re)acquire `data/raw/`'s contents (download script,
external source, generation command).

## Step 12 — `.env.example` and `CITATION.cff`

- Write `.env.example` with placeholders for secrets pipeline code will
  need (at minimum `WANDB_API_KEY=`; add `HF_TOKEN=` too if the project
  uses HF Hub — needed for the push in `data/make_dataset.py` regardless of
  the dataset's public/private visibility), and confirm `.env` itself is
  gitignored.
- Write a `CITATION.cff` at the repo root (`cff-version`, `title` from
  `project_name`, `authors` from `author_name`, `license` from the chosen
  `open_source_license`, `date-released` left as a placeholder) so the
  project is citable as academic software.

## Step 13 — `docs/adr/`

Create `docs/adr/` with a short `README.md` explaining that ADRs here are
for this *project's* expensive-to-reverse research decisions (model
architecture family, train/test split strategy, data representation) — not
routine implementation work. See `CLAUDE-CCDS.md`'s ADR section for the
full rationale.

## Step 14 — Extend the Makefile

Upstream `ccds` already generates `requirements`, `create_environment`,
`data`, `lint`, `format`, `test`, and `clean` targets. Add:
- `train` → runs `python -m <module_name>.modeling.train`
- `predict` → runs `python -m <module_name>.modeling.predict`

And fix the existing `data` target if it still points at the old
`<module_name>.dataset` path (Step 6 moved it to
`<module_name>.data.make_dataset`). Add `.PHONY: requirements
create_environment data train predict lint format test clean` at the top —
see `CLAUDE-CCDS.md`'s Orchestration section for why this matters (a stray
file sharing a target's name could otherwise make Make silently skip it).

## Step 15 — Cluster submission script

Write `scripts/submit_train.sbatch` per the template in `CLAUDE-CCDS.md`'s
"Cluster submission" section — `#SBATCH` resource values (account,
partition, GPU type, module names) are left as placeholders in the
generated file, not guessed at, since they're specific to whatever cluster
the user actually runs on. Fill in `<module_name>` in the job name and log
prefix. Leave the `WANDB_MODE=offline` line commented out by default
(wandb online is the default unless the user says their cluster's compute
nodes lack internet access).

## Step 16 — Finish up

1. `cd` into the new project directory.
2. `git init` if not already initialized, make an initial commit, then run
   `pre-commit install` so the hooks from Step 9 are active going forward.
3. If `environment_manager` isn't `none`, run `make create_environment` and
   `make requirements` so the project is immediately usable.
4. Show the generated tree (one or two levels deep) and point out:
   - `data/raw/` — original, immutable data, gitignored. Never edit in
     place; see `data/README.md` for how to (re)acquire it.
   - `data/interim/`, `data/processed/` — pipeline stages.
   - `notebooks/` — exploration only, numbered + initialed (e.g.
     `1.0-jqg-initial-eda.ipynb`); may import `src/<module_name>/` directly.
   - `src/<module_name>/{data,modeling,visualization}/` — the
     real, importable, tested subpackages, one per pipeline stage.
   - `models/`, `reports/figures/`, `references/`, `docs/`, `docs/adr/` — as
     named.
   - `.pre-commit-config.yaml`, `.github/workflows/ci.yml`,
     `.env.example`, `CITATION.cff`, `scripts/submit_train.sbatch` — the
     house tooling layer.
5. Remind the user that wandb needs `wandb login` (or `WANDB_API_KEY` in
   `.env`) before `make train` will work, and that
   `scripts/submit_train.sbatch` needs its `#SBATCH` placeholders filled in
   before it'll submit successfully on their actual cluster.

Do not silently skip step 16 — the point of using `ccds` plus this house
layer is a working, immediately-usable project, not just an empty folder
tree.
