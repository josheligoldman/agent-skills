# agent-skills

Claude Code skills for building repos in the
[cookiecutter-data-science](https://cookiecutter-data-science.drivendata.org/)
(ccds) style — scaffolding new ones, and reorganizing existing ones into it.

## Install

Globally, for every project (recommended for these two):

```bash
npx skills add -g <your-github-username>/agent-skills
```

Or just specific skills:

```bash
npx skills add -g --skill new-ds-project,refactor-ccds <your-github-username>/agent-skills
```

Update later with:

```bash
npx skills update
```

## What's included

- **[new-ds-project](skills/new-ds-project/SKILL.md)** — scaffolds a
  brand-new project with the real `ccds` CLI. Fetches `ccds`'s current config
  schema live from upstream on every run, so it can't silently go stale.
  Refuses on a non-empty target directory.
- **[refactor-ccds](skills/refactor-ccds/SKILL.md)** — reorganizes an
  existing, non-ccds repo into the ccds structure on a new branch, using
  `git mv` to preserve history. Runs a `grilling` session to resolve any
  file whose destination isn't obvious before moving anything.

See [CONTEXT.md](CONTEXT.md) for the vocabulary used across both skills, and
[docs/adr/](docs/adr/) for why the non-obvious design choices were made.

## CLAUDE.md conventions

These two skills handle scaffolding and one-time migration. To keep an agent
honoring the ccds structure during everyday work afterward — where new files
go, data immutability, notebook naming — copy this into your own
`~/.claude/CLAUDE.md` (global) or a project's `CLAUDE.md`:

```markdown
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
```

## License

MIT — see [LICENSE](LICENSE).
