<!--
codebase_evolution_summary.md

What this file does:
  A readable walkthrough of HOW the reproduced codebase physically changes as a
  paper moves through the main Paper2Code pipeline, organized by the five
  phase-groups in the workflow diagram. Each phase has a high-level "theory"
  section and a real before -> after code example taken from the `all-in-one`
  paper's two snapshots.

How to read it:
  Skim the "One-line mental model" first, then each "## <phase>" section.
  Theory = what the phase does; Example = the actual code that changed.

Source snapshots compared (the real evidence behind every example below):
  - outputs/paperbench_subrepos/all-in-one/stage3_11   (after coding)
  - outputs/paperbench_subrepos/all-in-one/stage8.1_11 (after iterative fixing)
-->

# How the Codebase Changes Through the Main Flow

The main pipeline is:

```
paper -> plan (1,2) -> codebase (3) -> codebase with dataset runner (5,5.1,6,8.2)
      -> iteratively-fixed codebase (7,8,8.1) -> codebase in docker (9a,9b,10)
```

> **One-line mental model:** the file *tree* is born in **codebase (phase 3)** and grown in the
> **dataset runner** phase. The **iteratively-fixed** phase adds almost no files — it *rewrites the
> insides* of existing files (stubs → real algorithms, configs → the paper's exact numbers). The
> `all-in-one` 3→8.1 diff is **0 net new source files, +252 lines, 19 files edited, 17 `.bak`
> backups**. **Docker** then seals it all into a container.

---

## Plan  (phase 1, 2)

### What this phase doing in theory
Reads the paper and produces *blueprints only* — **no source code is written yet**. It drafts the
overall reproduction plan, decides the architecture and which files will exist, sketches a
configuration template, and writes a per-file logic analysis. Every paper equation is extracted and
recorded so it can be checked once the code is written. The outputs are planning side files that
become the authoritative context for all later coding.

### Example what all-in-one code changes  (before -> after)
There is **no repo yet**, so the "change" is the blueprint that seeds it. The plan's config template
is what later becomes `config.yaml`:

```
BEFORE: (empty — no files exist)

AFTER (planning artifacts → seed config, e.g. the training budget the paper reports):
  # config.yaml (template from planning)
  task:
    name: lotka_volterra
    simulator: { sigma: 0.1, ... }
  # file task-list: main.py, src/simulators/lotka_volterra.py,
  #                 src/diffusion/guidance.py, src/baselines/*, ...
```

---

## Codebase  (phase 3)

### What this phase doing in theory
Turns the blueprint into **actual source files** — the whole planned codebase is written as
complete, runnable code (no TODO stubs): the main entry-point code file, the module files for each
planned component, a requirements file that pins the package versions, and a configuration file are
all generated here. Paper equations are implemented *literally* with hard-coded coefficients, and a
bounded self-verify can regenerate a file once if an equation is missing. This is the moment the
repo physically comes into existence.

### Example what all-in-one code changes  (before -> after)
A core module is born from nothing — e.g. the Lotka-Volterra simulator:

```python
BEFORE: (file does not exist)

AFTER: src/simulators/lotka_volterra.py
  """Lotka-Volterra simulator with irregular observations for Simformer.
  Paper-aligned properties implemented here:
  - 4 global parameters
  - classical predator-prey ODE dynamics
  - Gaussian observation noise with sigma=0.1 from `config.yaml`
  - irregular and species-specific observation times
  ...
  Implemented default reconstruction assumptions:
  - scaled sigmoid transform to (1, 3): 1 + 2 * sigmoid(z)
  - initial prey population = 1.0
  """
```
At the end of phase 3 the `all-in-one` repo already has ~163 files / ~61k lines.

---

## Codebase with dataset runner  (phase 5, 5.1, 6, 8.2)

### What this phase doing in theory
Takes the bare codebase and makes it *actually runnable against data and rival methods*. First it
produces an **evaluation-plan side file** that catalogues which datasets and baseline methods the
paper uses, each tagged with whether it fits the hardware/storage limits and is in scope. Guided by
that plan, it then grows the repo with the supporting machinery: data-preparation scripts, metric
code, dataset/baseline configuration files, and the real datasets and rival repositories downloaded
into an assets area. Finally it adds a set of baseline-runner modules — one per rival, behind a
shared dispatch table — so any competing method can be invoked through a single entry point. These
runners are wired as *stubs* first and smoke-tested, ready for the next phase to fill in.

### Example what all-in-one code changes  (before -> after)
A new dispatch registry appears, plus a per-rival adapter that is wired as a **stub first**:

```python
BEFORE: (no src/baselines/)

AFTER: src/baselines/__init__.py        # the runner dispatch table
  _BASELINE_SLUGS = ['nle', 'npe', 'npse', 'nre', 'simformer_posterior_only']
  def run_baseline_for(slug, dataset, config=None):
      module = importlib.import_module(f"src.baselines.{slug}")
      return module.run_baseline(dataset, config)

AFTER: src/baselines/nle.py  (initial STUB — falls back to bundled stack)
  def run_baseline(dataset, config=None):
      # The NLE "rival" is a stub; run the bundled Simformer stack
      from src.baselines.simformer_posterior_only import _run_repo_baseline
      overrides = dict(config or {}); overrides["mask_mode"] = "posterior"
      return _run_repo_baseline(dataset=dataset, config=overrides)
```
And the dataset catalogue is materialized as `configs/datasets.yaml` (here in its first,
*collection* form — see how the repair phase reshapes it next).

---

## Iteratively fixed codebase  (phase 7, 8, 8.1)

### What this phase doing in theory
Rubric-driven, **in-place repair** — the strongest "codebase changes" step. First it distills the
paper into a rubric: a checklist of testable requirements (is this method actually implemented? is
this exact hyperparameter present?). Then it walks that checklist against the code and **edits the
existing files in place** to fix every gap, keeping a numbered backup of each file it touches. It
never deletes code, and it works from the paper + rubric rather than the grading harness, so the
fixes generalize instead of overfitting the grader. The repo's file tree barely grows — almost all
the change is *content rewritten inside files that already exist* (in `all-in-one`: 0 new source
files, +252 lines, 19 files edited, 17 backup snapshots).

### Example what all-in-one code changes  (before -> after)

**(a) Baseline stub → real SBI implementation** (`src/baselines/nle.py`):
```python
BEFORE (stage3 — stub):
  def run_baseline(dataset, config=None):
      from src.baselines.simformer_posterior_only import _run_repo_baseline
      overrides = dict(config or {}); overrides["mask_mode"] = "posterior"
      return _run_repo_baseline(dataset=dataset, config=overrides)

AFTER (stage8.1 — real Neural Likelihood Estimation via sbi):
  def run_baseline(dataset, config=None):
      # Paper baseline: sbi training loop, neural spline flow (NSF),
      # batch size 1000, Adam, early stopping on validation loss.
      seed = int((config or {}).get("seed", 0))
      batch_size = int((config or {}).get("batch_size", 1000))
      from src.baselines.sbi_baselines import run_sbi_nle
      return run_sbi_nle(dataset=dataset, seed=seed, device=device,
                         batch_size=batch_size, config=config or {})
```

**(b) Config aligned to the paper's exact number** (`config.yaml`, the only line added):
```yaml
BEFORE: (no such key)
AFTER:  training_simulations: 100000  ## Section 4.2 / Figure 5 caption: 1e5 simulations
```

**(c) Dataset config reshaped: one collection → per-task** (`configs/datasets.yaml`):
```yaml
BEFORE (stage3 — single aggregated collection):
  sbibm_tasks:
    kind: "task_collection"
    prep:
      splits: ["train", "eval"]
      train_num_samples_per_task: 10000
      eval_num_samples_per_task: 10
      tasks: ["gaussian_linear", "gaussian_mixture", "two_moons", "slcp"]
    expected_paths: ["{benchmarks_dir}/sbibm_tasks/{train,eval}.pkl"]

AFTER (stage8.1 — one entry per task, split renamed eval→test, sizes bumped to paper scale):
  gaussian_linear:
    kind: "generated"
    simulator_task_name: "gaussian_linear"
    prep: { splits: ["train", "test"], train_num_samples: 20000, test_num_samples: 5000 }
    expected_paths: ["{benchmarks_dir}/gaussian_linear/{train,test}.pkl"]
  # gaussian_mixture / two_moons / slcp defined the same way ...
```

**(d) Missing paper method implemented** (`src/diffusion/guidance.py` — self-recurrence):
```python
BEFORE (stage3): "self-recurrence is intentionally not implemented"
  def __init__(self, scale_mode="sigma_inv_sq"): ...
  additive_constraint_score = self.constraint_score(x=x, t=t, constraints=constraints)

AFTER (stage8.1): optional multi-pass refinement (Lugmayr et al. 2022, paper App. A3.3)
  def __init__(self, scale_mode="sigma_inv_sq", self_recurrence_steps=None):
      self.self_recurrence_steps = int(self_recurrence_steps or 0)
  ...
  for _ in range(num_iters + 1):           # single-pass -> iterative refinement
      additive_constraint_score = self.constraint_score(x=x, t=t, constraints=constraints)
```

(Also: `scripts/prepare_data.py` gained download-**budget tracking** + `test_num_samples`, and
`tests/test_data_prep.py` assertions were updated to the per-task layout.)

---

## Codebase in Docker  (phase 9a, 9b, 10)

### What this phase doing in theory
First it **adds a unit-test suite** derived from the rubric: intermediate tests that check the
paper's key computations (shapes, invariants, reported intermediate numbers) and comparison tests
that check the reproduced method beats the wired baselines by the margin the paper claims, plus a
hook that records each test's pass/fail score. Tests that reference a symbol the code doesn't
actually have are commented out and skipped rather than deleted, so nothing silently breaks the run.
Then it **packages** the repo — without editing the source — into a self-contained container task:
a Dockerfile that bakes in Python, pytest, and the pinned dependencies, a task descriptor, a
verifier entry script, and the whole repo plus its downloaded assets copied into the image. The
result is an offline, gradeable bundle.

### Example what all-in-one code changes  (before -> after)
The repo is wrapped into a runnable container task (real artifacts at
`tests/harbor/all-in-one/`):

```
BEFORE: a bare repo folder (source + tests, run manually)

AFTER: harbor task bundle
  task.toml          # [verifier] timeout_sec=1800, env MODEL_NAME="claude-haiku-4-5"
  tests/test.sh      # cd /workspace; bash solution/solve.sh
                     # bash submission/reproduce.sh | tee reproduce.log
                     # python /tests/paperbench_verifier.py --paper-dir ... --submission-dir ...
                     # -> writes /logs/verifier/reward.txt
  Dockerfile         # python:3.11 + pytest + requirements baked in
  environment/       # the whole repo + assets/{benchmarks,rivals,metrics}
```
So the *code itself* is unchanged here — phase 9 bolts on the test suite, phase 10 freezes
everything into a Docker-runnable, offline, gradeable bundle.
