# `refactor-ccds` always branches, uses `git mv`, and grills before moving files

Reorganizing an existing repo's file layout is the kind of change that's easy
to get partially wrong in ways that aren't obvious until later — a script's
imports break, a notebook's relative paths shift, a file lands somewhere
that quietly contradicts what it actually does. `refactor-ccds` creates a
new branch before touching anything, uses `git mv` instead of a manual
move+delete so history follows each file, and — for any file whose
destination in the ccds structure isn't obvious from its name or location —
runs a `grilling` session with the user to resolve the mapping before
executing any moves, rather than pausing ad hoc mid-refactor or guessing
silently.

## Considered Options

- **Guess and flag afterward.** Faster, but wrong guesses compound — a
  misplaced module's imports get "fixed" downstream before anyone notices
  the module itself was misplaced.
- **Ask ad hoc, one file at a time, during the move.** Interrupts the
  process repeatedly and loses the chance to see the whole ambiguous set at
  once; a structured up-front session lets related ambiguities (e.g. three
  scripts that each mix feature-building and modeling) get resolved
  together instead of piecemeal.
