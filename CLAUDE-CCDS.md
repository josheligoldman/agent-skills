<!--
Copy everything below this line into your own ~/.claude/CLAUDE.md (global,
applies to every repo) or a single project's CLAUDE.md. It keeps an agent
honoring the cookiecutter-data-science (ccds) structure during everyday work
-- not just at scaffold time. The new-ds-project and refactor-ccds skills in
this repo handle the scaffolding/migration side; this is the ongoing-behavior
side.
-->

## Data science project conventions (cookiecutter-data-science)

This repo follows the cookiecutter-data-science (ccds) structure:
https://cookiecutter-data-science.drivendata.org/

- `data/raw/` is immutable. Never edit, overwrite, or regenerate files in it
  in place — treat it as read-only input.
- `data/interim/` and `data/processed/` hold pipeline outputs. These can be
  regenerated from `data/raw/` at any time.
- `notebooks/` is for exploration only. Name new notebooks
  `<number>.<owner-initials>-<short-description>.ipynb` (e.g.
  `1.0-jqg-initial-eda.ipynb`). Once logic in a notebook is reusable, promote
  it into `src/<module_name>/` — don't let notebooks accumulate real business
  logic.
- `src/<module_name>/` is the real, importable, tested package. New reusable
  functions belong here, organized into `data/`, `features/`, `modeling/`,
  and `visualization/` submodules matching ccds convention.
- `models/` holds serialized trained models and predictions. `reports/figures/`
  holds generated charts. `references/` holds data dictionaries and other
  explanatory material.
- Starting a brand-new data science/ML project? Use the `new-ds-project`
  skill rather than scaffolding by hand.
- Reorganizing an existing, non-ccds-structured repo into this layout? Use
  the `refactor-ccds` skill rather than moving files ad hoc.
