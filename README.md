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
go, data immutability, notebook naming — copy the contents of
[CLAUDE-CCDS.md](CLAUDE-CCDS.md) into your own `~/.claude/CLAUDE.md` (global)
or a project's `CLAUDE.md`.

## License

MIT — see [LICENSE](LICENSE).
