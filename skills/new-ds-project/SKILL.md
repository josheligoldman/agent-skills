---
name: new-ds-project
description: Scaffold a brand-new data science / ML project using the official cookiecutter-data-science (ccds) template, with config options fetched live from upstream every run. Use when the user asks to start a new data science, ML, or research project, wants a repo scaffolded "cookiecutter data science" style, or mentions ccds/cookiecutter-data-science. Refuses on a non-empty target directory — see the refactor-ccds skill for existing repos.
---

# New Data Science Project (ccds)

Scaffolds a new project using the real `ccds` CLI (the maintained successor to
plain `cookiecutter` for this template — see
https://cookiecutter-data-science.drivendata.org/), with its config schema
re-fetched live every run instead of hardcoded. See
[docs/adr/0001-live-schema-sync.md](../../docs/adr/0001-live-schema-sync.md)
for why: the schema has already changed shape once upstream, and a hardcoded
copy would silently go stale for everyone using this skill after the next
change.

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
  `testing_framework`, `linting_and_formatting`, `open_source_license`,
  `docs`, `include_code_scaffold`), the **first element is upstream's
  default**.
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
  testing_framework=<framework> \
  linting_and_formatting=<tool> \
  open_source_license="<license>" \
  docs=<docs> \
  include_code_scaffold=<Yes|No> \
  dataset_storage=none
```

If a field name from the live schema doesn't match what's shown above (the
template evolved further since this skill was written), adapt the command to
the field names you actually fetched — don't silently drop a field just
because it's unfamiliar.

If `--no-input` errors, fall back to running `ccds -o <parent_dir>`
interactively and answer each prompt on the user's behalf using the gathered
values, confirming with the user as you go.

## Step 6 — Finish up

1. `cd` into the new project directory.
2. `git init` if not already initialized, and make an initial commit.
3. If `environment_manager` isn't `none`, run `make create_environment` (or
   the equivalent) so the project is immediately usable.
4. Show the generated tree (one or two levels deep) and point out:
   - `data/raw/` — original, immutable data. Never edit in place.
   - `data/interim/`, `data/processed/` — pipeline stages.
   - `notebooks/` — exploration only, numbered + initialed (e.g.
     `1.0-jqg-initial-eda.ipynb`).
   - `src/<module_name>/` — the real, importable, reusable code.
   - `models/`, `reports/figures/`, `references/`, `docs/` — as named.

Do not silently skip step 6 — the point of using `ccds` is a working,
immediately-usable project, not just an empty folder tree.
