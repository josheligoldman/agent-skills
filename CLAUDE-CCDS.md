<!--
Copy everything below this line into your own ~/.claude/CLAUDE.md (global,
applies to every repo) or a single project's CLAUDE.md. It keeps an agent
honoring this project's cookiecutter-data-science (ccds)-based conventions
during everyday work -- not just at scaffold time. The new-ds-project and
refactor-ccds skills in this repo handle the scaffolding/migration side;
this is the ongoing-behavior side.
-->

## Data science project conventions (cookiecutter-data-science)

This repo follows the cookiecutter-data-science (ccds) structure
(https://cookiecutter-data-science.drivendata.org/), with this project's own
conventions layered on top of upstream's defaults where noted below.

- `data/raw/` is immutable. Never edit, overwrite, or regenerate files in it
  in place — treat it as read-only input.
- `data/interim/` and `data/processed/` hold pipeline outputs. These can be
  regenerated from `data/raw/` at any time.
- Actual data files are never committed to git — `data/` is gitignored
  (aside from `.gitkeep` placeholders). Document or automate how to
  (re)acquire `data/raw/`'s contents (a `README.md` in that directory, or a
  `make data` target that downloads/generates it) rather than relying on
  git to carry the files.
- `notebooks/` is for exploration only. Name new notebooks
  `<short-description>.ipynb` (e.g.
  `initial-eda.ipynb`).
- `models/` holds serialized trained models and predictions. `reports/figures/`
  holds generated charts. `references/` holds data dictionaries and other
  explanatory material.
- Starting a brand-new data science/ML project? Use the `new-ds-project`
  skill rather than scaffolding by hand.
- Reorganizing an existing, non-ccds-structured repo into this layout? Use
  the `refactor-ccds` skill rather than moving files ad hoc.

### `src/<module_name>/` layout: subpackages per stage

Unlike the current upstream ccds template (which generates flat files),
this project's convention is one **subpackage per pipeline stage**, so each
stage can grow past a single file without a disruptive reshuffle later:

```
src/<module_name>/
    __init__.py
    config.py            # paths, env loading, logging setup — see below
    data/
        __init__.py
        make_dataset.py   # raw -> interim/processed ingestion & cleaning
        validators.py     # pandera schemas, validated at pipeline boundaries
    modeling/
        __init__.py
        base.py            # Model Protocol + Embedder/Encoder Protocol —
                            # see below
        module.py          # LightningModule: wraps a Model with loss/
                            # optimizer/metrics — see "Training loop" below
        data_module.py      # LightningDataModule: batching/splits/num_workers
        train.py          # builds Model + LightningModule + DataModule,
                            # runs Trainer.fit
        predict.py         # loads a checkpoint, runs inference
        architectures.py   # concrete Model implementations + their Tyro
                            # configs, kept separate from the training loop
    visualization/
        __init__.py
        plots.py           # main entry point: data/results -> reports/figures
        <report_type>.py    # one file per report type as it grows
```

Split a file into more files under the same subpackage when it outgrows one
concern — don't dump unrelated helpers into one script, and don't invent a
different top-level shape than the one above. `architectures.py` is the
file most likely to hit this: once there are enough concrete `Model`
implementations that one file gets unwieldy, promote it to an
`architectures/` package — one file per architecture (`architectures/
random_forest.py`, `architectures/gnn.py`, ...), each still pairing its
`Model` implementation with its own Tyro config dataclass exactly as
before. `architectures/__init__.py` re-exports the unioned `ModelConfig`,
so `train.py`/`predict.py` import from `architectures` the same way either
way — promoting the file to a package is never a reason to touch the
stages that consume it.

There is deliberately no `features/` subpackage: every project here is
representation learning (GNNs, transformers, anything where the features
are learned rather than hand-engineered), so feature/embedding computation
is never a separate offline step — it happens inside the model itself, as
its own swappable component in `modeling/` (an `Embedder`/`Encoder`
`Protocol` in `modeling/base.py`, following the same decomposition pattern
as `Backbone`/`PredictorHead` below), computed during the forward pass.

### `config.py` is the single source of truth for paths

Every path a pipeline script touches — `RAW_DATA_DIR`, `INTERIM_DATA_DIR`,
`PROCESSED_DATA_DIR`, `EXTERNAL_DATA_DIR`, `MODELS_DIR`, `REPORTS_DIR`,
`FIGURES_DIR` — is derived from a `PROJ_ROOT` computed with `pathlib` in
`config.py`, plus `load_dotenv()` for anything secret/environment-specific
(API keys, wandb credentials). `.env` is gitignored; commit an `.env.example`
template instead. New code imports these constants; it never hardcodes a
relative path like `"../data/processed/x.csv"` or assumes a particular
working directory. `config.py` also configures `loguru` as the project's
logger (redirected through `tqdm.write()` when `tqdm` is present so progress
bars don't get mangled by log lines). Use `logger`, not `print`, in
pipeline code. Seeding uses Lightning's own `seed_everything()` (call it at
the start of `modeling/train.py` and log the seed value to wandb config) —
it already seeds `random`/`numpy`/`torch` and DataLoader workers
consistently, so there's no separate hand-rolled `set_seed()` to maintain.

### CLI: `Tyro`, everywhere, one tool for every stage

Every stage script exposes a CLI entry point guarded by
`if __name__ == "__main__":`, always via `tyro`. `tyro` supports two
shapes, and which one a stage uses depends on its parameter count:

- **Simple stages** (roughly ≤5-6 parameters, no dataset/model variants to
  swap) call `tyro.cli(main)` directly on a plain, typed function — `tyro`
  introspects the signature and generates the CLI from it. Parameters are
  typed (`Path`, `int`, `float`, ...) with defaults pointing at `config.py`
  constants, e.g. `input_path: Path = RAW_DATA_DIR / "dataset.csv"`.
- **Config-heavy stages** — typically `modeling/train.py`, and
  `modeling/predict.py` once it crosses that threshold — define a
  dataclass (or pydantic model) holding their parameters and call
  `tyro.cli(...)` on it instead of a plain function. The dataclass can be
  logged directly as `wandb.config`. When the stage picks between
  swappable implementations (see below), the config is a union of
  per-implementation dataclasses and `tyro` exposes the choice as a CLI
  subcommand.
- In both cases: narrate progress with `logger.info(...)` at the start of
  meaningful steps and `logger.success(...)` on completion; wrap iteration
  with `tqdm`; keep the entry point as an orchestration layer that calls
  into plain, testable functions rather than containing all the logic
  inline.

### Swappable implementations via `Protocol` interfaces

`modeling/` is built around a small interface plus
multiple concrete, swappable implementations from the start — not
introduced only once a second implementation shows up. Use
`typing.Protocol` (structural typing), not `abc.ABC`: a `Protocol` lets you
adapt a third-party class (a pretrained model wrapper) as a valid
implementation without subclassing anything. Reach for `abc.ABC` instead
only when implementations should also share concrete helper logic through
inheritance, not just conform to a method signature.

`Model` is forward-pass-only — it computes predictions from inputs, full
stop. Loss, optimizers, and metrics are a separate concern (see "Training
loop" below), never part of `Model` itself:

`modeling/base.py`:

```python
from typing import Protocol
from torch import Tensor

class Model(Protocol):
    def forward(self, batch) -> Tensor: ...
```

`modeling/architectures.py` holds each concrete implementation (a real
`nn.Module` subclass conforming to `Model`) paired with its own Tyro config
dataclass, exposed as a `tyro` union/subcommand so the implementation is
chosen straight from the CLI:

```python
from dataclasses import dataclass
from .base import Model

@dataclass
class MLPConfig:
    hidden_dim: int = 128
    num_layers: int = 3

    def build(self) -> Model:
        return MLPModel(self)

@dataclass
class GNNConfig:
    hidden_dim: int = 128
    num_layers: int = 3

    def build(self) -> Model:
        return GNNModel(self)

ModelConfig = MLPConfig | GNNConfig
```

`modeling/train.py` does `config = tyro.cli(ModelConfig)` and `model =
config.build()`, wraps `model` in the project's `LightningModule` (see
"Training loop" below), and hands that to a `Trainer`. `python -m
<module_name>.modeling.train mlp --hidden-dim 128` and `... gnn
--hidden-dim 256` both train through the same `Trainer`/`LightningModule`
regardless of which concrete `Model` was chosen. The analogous piece for
feature/embedding computation is an `Embedder`/`Encoder` `Protocol` in
`modeling/base.py` (raw input -> initial per-entity representation),
feeding into the backbone the same way.

#### Decompose into the actual swappable sub-parts

Before writing a stage's implementation, look for the components inside it
that are independently swappable, and give each its own `Protocol` — don't
stop at one interface per stage if the thing being built is itself made of
interchangeable parts. A model with a backbone and a predictor head is the
common case:

```python
# modeling/base.py
class Backbone(Protocol):
    def encode(self, features): ...

class PredictorHead(Protocol):
    def forward(self, embedding): ...

class Model(Protocol):
    def forward(self, batch) -> Tensor: ...
```

Concrete `Backbone`/`PredictorHead`/`Model` implementations are real
`nn.Module` subclasses (`Protocol` describes the shape structurally; it
doesn't replace `nn.Module` as the base class parameters need to be
tracked). The concrete `Model` is generic over its parts — composed via
dependency injection, not one hand-written subclass per backbone/head
combination:

```python
# modeling/architectures.py
class ComposedModel(nn.Module):
    def __init__(self, backbone: Backbone, head: PredictorHead) -> None:
        super().__init__()
        self.backbone = backbone
        self.head = head

    def forward(self, batch) -> Tensor:
        embedding = self.backbone.encode(batch)
        return self.head.forward(embedding)
```

and the Tyro config mirrors the composition — nested, independently
selectable unions rather than one flat union of whole models:

```python
@dataclass
class TrainConfig:
    backbone: GNNBackboneConfig | CNNBackboneConfig
    head: LinearHeadConfig | MLPHeadConfig

    def build(self) -> Model:
        return ComposedModel(self.backbone.build(), self.head.build())
```

so `train.py gnn-backbone --hidden-dim 256 mlp-head --num-layers 2` swaps
either part independently of the other, without a combinatorial explosion
of one concrete class per (backbone, head) pair.

This judgment call applies wherever it's real, not only to models: before
writing any stage's implementation, ask what the actual interchangeable
pieces are (backbone/head, tokenizer/encoder, whatever the domain calls
for) and give each its own `Protocol` in that subpackage's `base.py`,
rather than bundling them into one class that can only be swapped as a
whole.

Don't apply this pattern to `data/` or `visualization/` by default — those
stages usually have exactly one real implementation per project, and an
interface with a single implementation is speculative abstraction. Add a
`base.py` there too, following the same shape, only once a project actually
needs to swap data sources or renderers.

#### `common/` for genuinely shared code within a stage

When several concrete implementations in a stage share real, low-level
building blocks (shared layers, utilities, helpers used by more than one
`Backbone`/`Model`/`Embedder` implementation), put those in a
`common/` submodule inside that stage (e.g. `modeling/common/`) — not
duplicated per implementation, and not force-split across `data/`,
`modeling/`, `visualization/` just because the pieces happen
to be reusable. `common/` holds implementation details several
implementations depend on; it doesn't itself define a `Protocol` or get
swapped — if a "shared" piece turns out to actually need swapping, it's not
shared code anymore, it's another implementation of some `Protocol`.

### General code design habits

- **Single source of truth.** Hardcoding a value is fine — hardcoding the
  *same* value/shape in two or more places that can silently drift apart is
  the actual problem. If a quantity (a dimension, a config value, a
  derived constant) is needed in more than one place, derive it from one
  source — a property, a computed field, a shared constant it's imported
  from — rather than repeating the literal each place it's used.
- **Prefer one generic, parameterized implementation over many
  near-duplicate subclasses**, when the abstraction doesn't add much
  complexity to reach for it. Reserve a genuinely separate implementation
  (a new `Protocol`-conforming class) for behavior that differs *in kind*;
  when several variants differ only in configuration values, that's one
  class taking those values as constructor arguments, not one subclass per
  variant.
- **Dispatch on type, not on a string/enum tag.** When behavior needs to
  branch by "kind," express the kind as a distinct type (a subclass, a
  `Protocol` variant) and let normal Python dispatch (`isinstance`, method
  overriding) handle it, rather than a string/enum field checked with an
  `if`/`elif` chain that has to be kept in sync as kinds are added.
- **Keep a model's forward pass separate from training-loop concerns.** A
  `Model` implementation only computes predictions from inputs. Loss
  computation, optimizer/scheduler construction, metric logging, and any
  scaling/unscaling done purely for reporting belong in the project's
  `LightningModule` (`modeling/module.py`), which holds a `Model` instance —
  never inside the `Model` itself. See "Training loop" below.
- **Avoid logically-coupled optional constructor arguments in the first
  place** — if two optional parameters are related such that one implies
  the other, that's often a sign they belong together in one object or
  parameter instead of two independent ones. When it genuinely can't be
  avoided, validate the coupling explicitly at construction time with a
  clear error naming both arguments, rather than letting a
  half-configured object fail later in a confusing way.
- **Use `nn.ModuleList`/`nn.ModuleDict` for any list/dict of submodules**
  on an `nn.Module`. A bare Python list or dict attribute isn't tracked by
  PyTorch, so those submodules' parameters silently vanish from
  `.parameters()`, never move with `.to(device)`, and never appear in a
  checkpoint.

### Pipeline shape: raw -> interim/processed -> model -> predictions/reports

There's no separate feature-engineering stage: `data/make_dataset.py`
produces cleaned structures/examples, and `modeling/train.py` consumes
them directly through its `Embedder`/`Encoder` component — feature
computation happens inside the model, not as an intermediate pipeline
step.

- `data/make_dataset.py` never mutates `data/raw/`; it reads from there and
  writes cleaned output to `data/interim/` or `data/processed/`, validated
  against a `pandera` schema in `data/validators.py`.
- `modeling/train.py` builds a `Model` + `LightningModule` + `LightningDataModule`
  from `data/processed/`, seeds reproducibility via `seed_everything()`, and
  runs `Trainer.fit(...)` with a `WandbLogger` (source of truth for run
  history/versioning) *and* a `ModelCheckpoint` callback writing to
  `models/` (e.g. `models/latest.ckpt`) so `predict.py` and local work
  don't require network access. See "Training loop" below.
- `modeling/predict.py` loads a checkpoint from `models/`, runs inference
  against processed data from `data/processed/`, and writes predictions to
  `data/processed/` or `reports/` — it never re-trains.
- `visualization/plots.py` reads processed data / predictions and writes
  figures to `reports/figures/`; it never touches `data/raw/` or `models/`
  for writing.
- Every stage should be safe to re-run from `data/raw/` onward and produce
  the same output (modulo the seeded randomness) — if a change makes a
  stage non-reproducible (relies on notebook-only state, an untracked
  global, a manual step), treat that as a bug to fix, not something to
  route around.

### Training loop: PyTorch Lightning

Lightning is the standard training-loop framework — it's the concrete
mechanism behind "keep a model's forward pass separate from training-loop
concerns" above, not a per-project choice.

- `modeling/module.py` holds one project-specific `LightningModule`
  subclass (e.g. `<ModuleName>Module`) wrapping a `Model` instance plus
  loss, optimizer/scheduler construction (`configure_optimizers`), and
  metric logging (`training_step`/`validation_step`/`test_step`). It's
  generic over whichever concrete `Model` was chosen — the same wrapper
  regardless of `MLPModel` vs `GNNModel`.
- `modeling/data_module.py` holds a `LightningDataModule` handling splits,
  batching, `num_workers`, and any collation specific to the model's input
  representation, built from `data/processed/`.
- `modeling/train.py` wires these together: `model = config.build()` (from
  the Tyro `ModelConfig` union) → `module = ProjectModule(model, ...)` →
  `data_module = ProjectDataModule(...)` → `trainer = Trainer(logger=
  WandbLogger(log_model="all"), callbacks=[ModelCheckpoint(dirpath=
  MODELS_DIR)], ...)` → `trainer.fit(module, datamodule=data_module)`.
- `WandbLogger(log_model=...)` automatically logs checkpoints as wandb
  Artifacts — this is what satisfies the wandb-Artifact-plus-local-copy
  dual-write requirement, without hand-rolled upload code.
  `ModelCheckpoint` writes the local copy `predict.py` reads.
- `modeling/predict.py` loads a checkpoint via
  `ProjectModule.load_from_checkpoint(path)` and runs inference directly
  (or via `Trainer.predict(...)` for a large input set) — it never calls
  `Trainer.fit`.

### Publishing the regenerated dataset to HuggingFace Hub

If a project uses HF Hub for datasets, `data/make_dataset.py` pushes the
resulting processed dataset there by default, as part of regenerating it —
not a separate manual publish step. This mirrors the dual-write pattern
already used for models: local `data/processed/` is what everything
downstream actually reads; HF Hub is the remote copy, for sharing/backup/
reproducibility, that nothing in this pipeline ever reads back from.

- The push is on by default, but overridable (`--no-push`, or a
  `push: bool = True` param) — disable it for local iteration while still
  tweaking the cleaning logic, so every experimental regeneration doesn't
  land on the HF repo.
- **`modeling/train.py` and `modeling/predict.py` never import `datasets`
  or touch HF Hub at all** — they only ever read `data/processed/` locally.
  No pull-from-HF fallback logic, ever; that would reintroduce the network
  dependency at training time this design specifically avoids.
- `HF_TOKEN` in `.env`/`.env.example` is needed for the push regardless of
  the dataset's public/private visibility — pushing always requires write
  auth, unlike pulling from a public dataset, which needs none (moot here
  anyway, since nothing pulls).

### Orchestration: `Makefile`

Stages are chained via `Makefile` targets, not a dependency-tracked
pipeline tool (no DVC/Snakemake by default — add one only if a specific
project genuinely needs hash-based rerun tracking or remote data storage).
Canonical targets:

```
make requirements        # install/sync dependencies
make create_environment  # create the virtualenv/conda env
make data                # data/make_dataset.py
make train                # modeling/train.py
make predict              # modeling/predict.py
make lint                  # ruff check
make format                 # ruff format
make test                    # pytest
make clean                    # remove data/interim, data/processed, reports/figures
                               # so the pipeline can be rerun from data/raw/
```

Declare every target `.PHONY` (`.PHONY: requirements create_environment data
train predict lint format test clean`). Without this, Make's default
timestamp-based logic could decide a target is "already up to date" if a
stray file or directory ever happens to share its name (`train`, `data`,
...) — and silently skip the recipe instead of erroring. `.PHONY` makes
every target always run regardless of file state.

### Cluster submission: `scripts/submit_train.sbatch`

Training runs on a SLURM cluster via a checked-in `sbatch` template that
wraps `make train`:

```bash
#!/bin/bash
# Usage: sbatch scripts/submit_train.sbatch [tyro args...]
# e.g.: sbatch scripts/submit_train.sbatch gnn --hidden-dim 256
#
# #SBATCH pragmas are parsed before this script runs, so they can't see the
# args above -- the job starts under a generic name/log path, then renames
# itself once the args are known, so squeue and the log filename reflect
# what's actually running (and resubmissions stay unique).
#SBATCH --account=<ACCOUNT>            # set for your cluster/allocation
#SBATCH --partition=<PARTITION>        # set for your cluster
#SBATCH --gres=gpu:1                   # adjust/remove if not using a GPU
#SBATCH --cpus-per-task=16
#SBATCH --mem=128G
#SBATCH --time=6:00:00
#SBATCH --job-name=<module_name>_train
#SBATCH --output=%x-%j.out.tmp         # replaced below once args are known

set -euo pipefail

JOB_NAME="<module_name>_train_$(echo "$*" | tr ' /' '__')_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$SCRATCH/logs"
exec > "$SCRATCH/logs/${JOB_NAME}-${SLURM_JOB_ID}.out" 2>&1
scontrol update JobId="$SLURM_JOB_ID" JobName="$JOB_NAME" || true

module load <MODULES>          # e.g. python/3.11 cuda/12.6 — set for your cluster

# Caches that can grow large go on scratch, not a quota-limited home dir.
export HF_HOME="$SCRATCH/hf_cache"
export UV_CACHE_DIR="$SCRATCH/uv_cache"     # or PIP_CACHE_DIR / CONDA_PKGS_DIRS,
                                             # matching your environment_manager

# Uncomment if you hit GPU memory fragmentation on large models:
# export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True

# wandb defaults to online (needs the compute node to reach the internet).
# If your cluster's compute nodes lack outbound network access, uncomment
# the next line and sync afterward with `wandb sync <run-dir>` from a login
# node or your own machine instead:
# export WANDB_MODE=offline

source .venv/bin/activate      # or `uv run`/`conda activate ...`, matching
                                # your environment_manager

make train ARGS="$*"
```

- The `#SBATCH` values (account, partition, GPU type, module names) are
  inherently cluster-specific — left as placeholders, not guessed at,
  since this convention can't know your cluster's actual configuration.
- `ARGS="$*"` forwards whatever's passed to `sbatch` straight through to
  the Tyro CLI: `sbatch scripts/submit_train.sbatch gnn --hidden-dim 256`
  runs that exact model variant on the cluster, and the job-rename logic
  bakes those same args into the job name and log filename.
- Logs and caches (`HF_HOME`, the package manager's cache) go on
  `$SCRATCH`, not the repo or home directory — home filesystems on
  clusters are often quota-limited; scratch is meant for exactly this kind
  of large, regenerable data.
- Whether wandb runs online or offline depends on whether the cluster's
  compute nodes have outbound internet access — this varies per cluster,
  so the template defaults to online (wandb's own default) and documents
  the offline path as a comment rather than assuming either way.

### Testing

Every plain function underlying a stage (not just the CLI wrapper) gets a
`pytest` test in `tests/`, mirroring the `src/<module_name>/` subpackage
structure — one test file per source file. Testing the `tyro` CLI wrapper
itself (e.g. calling `tyro.cli(main, args=[...])` with an explicit args
list rather than reading `sys.argv`) is nice-to-have, not required — the
logic worth testing lives in the functions the wrapper calls.

- **Shared fixture builders, not copy-pasted setup.** Put reusable,
  deterministic, hand-built tiny fixtures (a minimal batch, a minimal
  config object) in `tests/_helpers.py` or a `conftest.py` at the level
  that needs them, parameterized with sensible defaults — not
  re-constructed inline in every test file that needs one.
- **Small hand-built synthetic data for unit tests**; a tiny *real* data
  sample bundled under the relevant `tests/<subpackage>/data/` for
  pipeline-level smoke tests. Since `data/raw/` itself is gitignored and
  unavailable in a fresh clone or CI, a pipeline/integration test that
  needs to exercise real parsing edge cases needs its own small, checked-in
  sample — separate from, and much smaller than, the real dataset.
- **Conformance tests across every swappable implementation.** Any
  `Protocol` with more than one concrete implementation (`Model`,
  `Backbone`, `Embedder`, ...) gets one parametrized test suite run against
  every implementation/config variant, rather than a bespoke test file per
  implementation duplicating the same assertions — this is also the right
  place to assert that a config knob actually changes behavior (e.g. a
  `num_layers` override actually changes how many layers get built), not
  just that it's accepted without error.
- **Test constructor invariants you added.** Any explicit validation from
  the "avoid logically-coupled optional constructor arguments" rule above
  gets a test asserting it actually raises, with a message naming the
  right argument(s) (`pytest.raises(ValueError, match="...")`).
- **Skip cleanly around optional heavy dependencies.** Use
  `pytest.importorskip` for tests that depend on an optional/heavy extra
  (a specific ML framework, GPU-only code) so an environment without that
  extra installed skips those tests instead of failing the whole suite or
  forcing every environment to install every optional dependency.
- **Test `pandera` schemas directly**: a valid row/frame passes, an invalid
  one is rejected — for every schema in `data/validators.py`.
- **Use tolerance-aware numeric comparisons** (`numpy.testing.assert_allclose`,
  `torch.testing.assert_close`) for floating-point results, never bare
  `==`.
- **Don't test third-party library internals** — trust that `sklearn`,
  `torch`, etc. work; test your own glue/wrapper code around them.
- **Use `hypothesis` for functions with a statable invariant**, alongside
  (not instead of) example-based `pytest` tests — scalers, outlier
  filters, and other data transforms are the common case (e.g. "inverse
  undoes forward" for a scaler: `@given(st.floats(...))` generates many
  edge-case inputs trying to break that property, rather than relying on
  the handful of examples a human happens to write by hand). Reach for it
  when a function's correctness is naturally a property over a range of
  inputs; don't force it onto code that's just a fixed sequence of steps
  with no such invariant.
- **`pytest-cov`, report-only, no hard threshold.** Run with `--cov` in CI
  so coverage is visible, but nothing gates on a percentage — this is an
  academic repo, not a shop enforcing a coverage SLA. Its most useful job
  is validating a `hypothesis` strategy: if a `@given`-decorated test's
  target branch never shows up as covered despite many generated examples,
  the strategy is too narrow (missing an input class like negative values
  or the near-zero case), not the branch being untestable.

### Code quality tooling

- **`ruff`** for linting and formatting, including its docstring rules
  (`D` ruleset) enforcing **NumPy-style docstrings** — the convention
  already used by numpy/scipy/pandas/sklearn — and its McCabe complexity
  rule (`C901`, `max-complexity = 10` in `[tool.ruff.lint.mccabe]`),
  flagging functions whose branching has gotten hard to reason about
  (common in outlier handling, config branching, and other conditional
  data-cleaning logic) so they get split up before they become a problem.
- **`mypy`** runs in CI (not commit-blocking) — see the dedicated policy
  below for how strict it actually is.
- **`pandera`** for DataFrame schema validation in `data/validators.py`,
  used at each raw→interim→processed boundary.
- **`pre-commit`** runs `ruff` (lint + format), `mypy`, and `nbstripout`
  (strips notebook outputs/metadata) and **blocks local commits** — install
  it as part of project setup, don't leave these checks CI-only.
- **GitHub Actions CI** runs tests + lint + type-check on every push/PR.
- **`pip-audit`** runs in CI to flag known-vulnerable dependencies.
- A **`CITATION.cff`** at repo root makes the project citable — standard
  for academic research software.

### `mypy` policy: strict on your code, contained at the third-party boundary

Chemistry/ML libraries (RDKit, pymatgen, ASE, and others in that space) are
often unstubbed or only partially typed, so one global strictness setting
can't serve both "hold our own code to a high bar" and "don't fight
libraries we don't control." Apply strictness unevenly, on purpose:

- **Your own code** (`src/<module_name>/**`) is strict:
  `disallow_untyped_defs`, `disallow_incomplete_defs`, `check_untyped_defs`,
  `no_implicit_optional`, `warn_return_any`, and `warn_redundant_casts` all
  on. Every function gets a full signature — this is what "the codebase is
  typed throughout" actually cashes out to.
- **Untyped third-party libraries get named, per-module overrides** —
  `[[tool.mypy.overrides]]` blocks with `module = ["rdkit.*", "pymatgen.*",
  "ase.*"]` (adjust to whatever's actually unstubbed in a given project)
  setting `ignore_missing_imports = true` for just those modules. Never set
  `ignore_missing_imports = true` globally — that would silently excuse gaps
  in your *own* code too, not just the third-party import.
- **Contain the untyped boundary in one wrapper, don't let `Any` leak.**
  Where an untyped third-party value first enters your code (an RDKit
  `Mol`, an ASE `Atoms` object), wrap the call in a small, explicitly
  return-typed function — converting it into one of your own typed
  structures (a dataclass, a `TypedDict`) immediately — rather than passing
  the untyped object several calls deep before anything gets typed. One
  annotated seam, not `Any` silently propagating through half the pipeline.
- **`# type: ignore[specific-code]`, never a bare `# type: ignore`.** Scope
  every suppression to the exact error code; add a short comment when it's
  suppressing a real type mismatch (not just a missing-stub warning) so a
  future reader knows it was a deliberate call, not an oversight.
- **`warn_unused_ignores = true`** — flags an `# type: ignore` that's no
  longer doing anything, e.g. once a library finally ships type stubs and
  the suppression becomes dead weight instead of a documented exception.

### ADRs for project-level decisions

Alongside this skills repo's own `docs/adr/` (which documents decisions
about the skill tooling itself), a scaffolded data-science project gets its
own `docs/adr/` for that project's expensive-to-reverse research decisions
— e.g. choice of model architecture family, train/test split strategy
(random vs. scaffold/group split), or data representation (e.g. SMILES
strings vs. 3D molecular graphs). Write an ADR only for calls like these,
not for routine implementation work.

An agent never makes one of these calls unilaterally — surface it to the
user as an explicit question (options, tradeoffs, a recommendation) and
get their answer first. The ADR then records what was decided and why;
it documents a decision that was made in consultation with the user, not
a decision the agent made on its own and is retroactively explaining.

### Notebooks and `src`

A notebook may `import <module_name>` and call existing `src` functions
directly — it isn't limited to a black box. New or exploratory logic can
stay inline in the notebook temporarily, but must be promoted into `src`
(with a test) before it's reused a second time or relied on by another
pipeline stage. Don't let notebooks accumulate real business logic long
-term.

### Code style and comments

- Docstrings are required on every public function/class in
  `src/<module_name>/`, NumPy-style (per the ruff `D` rules above) — a
  one-line summary plus Parameters/Returns. This is shared research code,
  not scratch notebook cells, so it needs to be usable by collaborators and
  future-you.
- Private functions get no docstring by default — a well-named private
  helper next to its one call site doesn't need one. But when a private
  function's behavior or reasoning is genuinely non-obvious (nontrivial
  math, a subtle algorithm, a non-obvious constraint), add a short
  explanation — not the full NumPy Parameters/Returns ceremony, just enough
  to cover the specific thing that isn't obvious. A one-line docstring or
  an inline comment both work; use whichever fits.
- Inline comments explain *why*, not *what* — a non-obvious constraint, a
  workaround for a specific data quirk, the reason a threshold or
  hyperparameter was chosen. Don't narrate what the code does; well-named
  functions and variables already do that.
- No commented-out code or context-free TODOs in committed code — finish
  it, delete it, or turn it into a tracked issue/ADR if it's a real open
  decision.

### Git & GitHub: foundational practices

This is an academic research context, not a software-engineering team —
keep this to git/GitHub hygiene, not a full branch-protection/PR-mandatory
workflow:

- Commit early and often, with clear, imperative-mood messages that explain
  *why* a change was made, not just what changed.
- Use a separate branch for anything experimental or risky (a big refactor,
  an architecture change you might abandon) so `main` stays in a working
  state; merge or discard the branch once you know how it turned out.
- Push to GitHub regularly — it's the project's backup and shareable
  record, not just local history.
- Never force-push or rewrite history on `main`, or on any branch someone
  else might be building on.
- No PR requirement, no commit-message prefix scheme, no branch-protection
  rules by default — that's more process than a research repo needs.
  Reconsider this section only if a project grows enough active
  contributors that "don't lose work, be able to explain a decision later"
  (which ADRs also help with) stops being enough on its own.
