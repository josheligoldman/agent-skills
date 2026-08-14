# Always fetch ccds's live schema instead of hardcoding it

`new-ds-project` needs to know `ccds`'s current config options (environment
manager, dependency file, license, etc.) to scaffold non-interactively. It
fetches `ccds.json` from upstream (`drivendataorg/cookiecutter-data-science`)
at the start of every run instead of keeping a hardcoded copy of the schema
in the skill, because the schema has already changed shape once before
(plain `cookiecutter` → the `ccds` CLI, with a reshaped config) and a
hardcoded copy would silently go stale for anyone using this skill after the
next change.

## Considered Options

- **Hardcode the schema.** Simplest, no network dependency — but drifts
  silently when upstream changes. The failure mode is either a cryptic error
  from `ccds --no-input` on an unrecognized field, or worse, defaults that
  are just wrong with no warning to the user.
- **Hybrid: hardcode as a fast path, live-fetch only to check for drift.**
  Safer than pure hardcoding, but this repo is shared across a whole lab —
  the cost of one extra fetch per scaffold is small compared to the risk of
  someone hitting a schema mismatch the sanity-check didn't happen to catch.
  Always-live removes that risk entirely rather than reducing it.
