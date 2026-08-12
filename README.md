# stegorobust — revision code for IJIES paper 20264187

Code accompanying the revision of *"Test-time Evasion of Deep Steganalysis
Committees and its Transferability."* Every module here exists because of a
specific reviewer comment; `REVIEWER_MAP.md` is the index from comment to code.

The short version of what changed:

1. **The detectors now converge, or they are excluded.** Each of XuNet, YeNet and
   SRNet gets its own published training recipe instead of one shared 100-epoch
   budget, models are selected on best validation AUC rather than the last epoch,
   and a `ConvergenceGate` refuses to let a chance-level detector into the
   committee or into the held-out role.
2. **The transferability gap is guarded.** It is computed only when both the
   attacked committee and the held-out detector pass the gate. On the submitted
   run it would return `None` with a written reason — which is the honest result.
3. **Score polarity is fixed a priori**, so an AUC below 0.5 means an inverted
   ranking rather than an unexamined sign convention.
4. **Comparisons are matched and head-to-head**: ten attacks (through NAA, CVPR
   2022) at identical budget, eight embedders at identical payload.
5. **Uncertainty is reported**: DeLong CIs on every AUC, paired tests across
   seeds, Holm correction within each table, and a claim gate that fails any
   assertion whose margin sits inside the seed-to-seed spread.

---

## Install

```bash
pip install -r requirements.txt
```

Reproducing the manuscript's generator additionally requires the released
SteganoGAN weights. The wrapper raises rather than substituting a different
generator, because a silent substitution would put a different model behind the
same name in the results table.

## Run

**Running the actual revision:** open `notebooks/run_revision_colab.ipynb` in Colab
and work top to bottom. It gates Phase 2 behind the SRNet convergence verdict, so
you cannot spend two days of GPU on a configuration that cannot answer the
reviewers. The commands below are what it runs.

```bash
# 0. sanity: whole pipeline on synthetic data, CPU, no downloads
python scripts/smoke_test.py

# 1. covers, crops, source-disjoint split
python scripts/prepare_data.py dataset=bsds500

# 2. detectors, per-detector recipes, best-val checkpointing, convergence gate
python scripts/train_detectors.py -m seed=0,1,2,3,4,5,6,7

# 3. evasion sweep (the manuscript's PGD, then the rest of the suite)
python scripts/run_attack.py -m attack=pgd,naa seed=0,1,2,3,4,5,6,7

# 4. matched-payload embedding comparison
python scripts/run_embedding_comparison.py embed=comparison

# 5. ablations -- each row changes exactly ONE field (configs/ablation/)
python scripts/train_detectors.py -m \
    ablation=full_model,no_paired_batching,frozen_srm,no_zero_init,no_curriculum,no_augment \
    seed=0,1,2,3
python scripts/run_attack.py -m \
    ablation=committee_mean,committee_minmax,committee_logit,no_quantize seed=0,1,2,3

# 6. hyperparameter search -- validation only, pruned, resumable
python scripts/tune.py --detector srnet --trials 25 --epochs 120

# 7. figures (PDF/SVG) and booktabs tables (.tex)
python scripts/aggregate_results.py --metrics outputs/metrics --out outputs

# 6. manuscript audit: acronyms, PSNR claims, cross-table AUC agreement
python tools/check_manuscript.py "Shihab et al..docx" --json audit.json

# optional: version-controlled runs and autonomous supervision (both off by default)
python scripts/train_detectors.py git.enabled=true git.remote=https://github.com/<owner>/<repo>.git
python scripts/run_supervisor.py --wandb-run entity/project/run_id --repo <clone> --once
```

Multi-dataset and multi-payload sweeps are Hydra overrides, not code changes:

```bash
python scripts/train_detectors.py -m dataset=bsds500,bossbase,div2k,coco seed=0,1,2
```

## Weights & Biases

Store the API key in Colab Secrets as `WANDB_API_KEY` (from <https://wandb.ai/authorize>),
never in a cell — saved notebook output is not retractable.

All three detectors log into **one run**, namespaced by detector
(`srnet/val_auc`, `yenet/val_auc`, …). The supervisor filters on that prefix, so
**do not flatten these names**: stripping the prefix merges three series into one
and a watcher following SRNet silently reads YeNet's curve. `Supervisor.parse_history`
keeps the full key when no detector is named rather than guessing which series wins.

The training script prints the run path (`<entity>/<project>/<run_id>`) and the exact
`run_supervisor.py` command to paste, because hunting for a run id in the UI mid-training
is the friction that stops people using the watcher at all.

Notebook cell 0.6 verifies the round trip — logs a throwaway run in the real key format,
reads it back through the supervisor's own code path, and asserts the correct trigger
fires. Run it before Phase 1, not after.

## Ablations

Every novel component is a YAML toggle, so an ablation is a configuration change
rather than a code branch, and each file in `configs/ablation/` changes **exactly
one** field. Three of them test claims the response letter makes in writing:

| Ablation | Claim it tests |
|---|---|
| `no_paired_batching` | That unpaired batching is why XuNet collapsed to a constant prediction — demonstrated, not asserted |
| `no_curriculum` | That the payload curriculum lifts SRNet off chance, as distinct from the longer schedule |
| `committee_minmax` | That the manuscript's mean-of-losses objective understates a committee attack |

## Hyperparameter search

`scripts/tune.py` runs Optuna with a median pruner, a TPE sampler, and the study
persisted to Drive so an interrupted search resumes. The space is deliberately
narrow and anchored on the published recipes: this is a revision, and every
decimal place tuned on validation is one the reported test number will not
reproduce. The test split is never constructed in that file, and
`assert_no_test_split` guards the objective's scope.

## Version control and provenance

One repository per paper; the history is the record of how the result was reached.

**The token is never handled directly.** Store a fine-grained PAT in Colab
Secrets, scoped to this repository, `Contents: read/write` only, expiry ≤ 90 days:

```python
from google.colab import userdata
os.environ["GITHUB_PAT"] = userdata.get("GITHUB_PAT")
```

It is passed per-invocation through a credential helper, never written into the
remote URL — `git remote set-url` persists it in `.git/config`, which leaks the
moment the directory or VM image is shared. `assert_token_not_persisted` scans
`.git/config` and the remote before every push and raises if it finds one; a
token that reaches that state must be revoked, not deleted.

If you are asked to paste a token into a chat, don't. Message history cannot be
reliably retracted.

**Commit policy.** One commit when validation improves on the previous best — not
every epoch, not every checkpoint — plus exactly one end-of-run commit carrying
evaluation artifacts (`.tex` tables, vector plots, profiling), which are produced
after training stops and would otherwise never land. Messages carry full
provenance so any number in the manuscript traces to an exact state:

```
best: [yenet] val_auc 0.8412 (prev 0.8377) @ epoch 37

seed: 42
config_hash: a3f9c1e
wandb_run: 3kx9m2p1
trigger: new_best_val
agent_revision: 2/3
```

`best_model.pt` is the only weight file in the repository, via Git LFS.
Intermediate checkpoints stay on Drive and are referenced by path — free-tier LFS
quotas are small enough that one careless glob exhausts them.

## Autonomous training supervision

Opt-in (`supervisor.enabled=true`). An agent that edits your experiment should
never be on by default.

The supervisor runs **outside** the runtime. It polls W&B, decides, and pushes a
patch to the `agent/patches` branch; a thin stub inside the notebook polls that
branch between epochs and applies what it finds. Watcher decides, stub applies —
nothing here can restart a Colab VM, and anything claiming otherwise is fiction.

```bash
python scripts/run_supervisor.py --wandb-run entity/project/run_id \
    --repo /content/drive/MyDrive/stegorobust --detector srnet --once --dry-run
```

**Dual-target.** One supervisor manages local and hosted runs from a single
registry keyed by experiment ID. The decision logic is shared; only delivery
differs — `direct` (patch and relaunch on the same machine) for `local`, the
`agent/patches` branch for hosted targets. The three-round budget is **shared
across targets**, tracked against the experiment ID rather than the process, or a
migration silently resets it. The budget is not relaxed locally just because
restarts are cheap: the cap bounds validation overfitting, not compute.

**Migration** fires on sustained host pressure defined numerically (load per
core, memory %, swap %, held for N consecutive minutes — a spike is not
pressure). The checkpoint carries full RNG state, so **metrics survive a
migration and timings do not**: the run is marked `timing_invalid`, its latency
and epoch-time are excluded from every table, and re-running on fixed hardware is
required if that measurement is needed.

Trip conditions are numbers, not judgment calls (`configs/supervisor/default.yaml`):
divergence (NaN, or loss above 10× its running median) fires immediately;
plateau, overfitting, and underperformance-against-a-named-baseline are subject
to a minimum interval between interventions.

The `underperformance` trigger is the one that matters for this paper. The
submitted study did not fail by diverging — it failed by sitting at chance for a
hundred epochs while training loss fell. Its baseline is the convergence gate
itself (clean AUC 0.75), so that failure is caught at epoch 20 rather than at the
end. SRNet gets a longer warmup, because it is *expected* to sit near chance
early in its 450-epoch schedule and the generic threshold would fire on healthy
training.

Three constraints, all tested:

- **The supervisor never sees the test split.** `assert_no_test_split` inspects
  the W&B history and the run state and raises on anything test-shaped. This is
  structural, not a prompt instruction: an agent that iterates against a metric
  is a search procedure, and any split it can observe stops being evaluation.
- **The budget is three rounds, and the counter lives in the checkpoint.** In
  memory it would silently reset on a Colab reconnect and the cap would mean
  nothing. After three rounds the supervisor halts and notifies.
- **Patches are bounded.** Only allow-listed config keys may be overridden, and
  an equivalent patch is never re-applied.

Edits are two-tier. *Mechanical* ones (learning rate, clipping, optimizer,
regularization) are applied silently. *Method-altering* ones are also applied
automatically but tagged, and `train_detectors.py` prints a consolidated summary
at the end of the run:

```
METHODS SECTION WARNING
1 method-altering intervention(s) were applied automatically during training.
The methods section must describe the code as it ran, not as it was written:
  round 1 (underperformance): enforce paired batching, freeze the SRM front end … [0.5000 -> 0.9400]
```

Every intervention appends one line to `outputs/agent_interventions.jsonl`,
committed with the run. That file is the answer to "how did you arrive at this
architecture," and it stays accurate even when it is unflattering.

## Execution targets (local vs hosted)

The target is **declared, never inferred** — a run that silently changes hardware
is a run whose timings mean nothing:

```bash
python scripts/run_embedding_comparison.py env=local       # default
python scripts/train_detectors.py           env=colab_gpu
```

Routing in this project:

| Work | Target | Why |
|---|---|---|
| Embedding comparison (HILL, S-UNIWARD, ternary simulator, LSB) | `local` | Pure NumPy/SciPy, entirely CPU-bound — the largest CPU cost here |
| Detector training (XuNet, YeNet, SRNet) | `colab_gpu` | 256×256, and SRNet runs 450 epochs |
| Any number that enters a results table | `local` | Fixed hardware |
| Memory-heavy exploration, demo notebook | `colab_cpu` | Timings auto-marked non-reportable |

Local is the default not because it is faster but because a hosted runtime
allocates a different VM each session: the CPU model varies run to run.
**Numerics survive once threads are pinned; timings do not.**

**Determinism is harder on CPU, not easier.** BLAS reduction order changes with
thread count, so the same seed at a different thread setting gives different
floats. `configure()` pins `OMP`/`MKL`/`OPENBLAS`/`NUMEXPR`/`VECLIB` and
`torch.set_num_threads` before any tensor exists, and the thread count is
recorded next to the seed — it is part of the reproducibility contract, not a
performance note. `sklearn_n_jobs()` refuses `-1` for a pinned run, since that
makes results depend on the host's core count.

Pinning costs real parallelism, so it is a *mode*, not a global.
`thread_mode: pinned` for anything reported; `exploration` for everything else,
which marks the run's timings non-reportable automatically. A timing measured
multithreaded and a metric measured single-threaded must never share a row.

**Apple Silicon:** MPS is refused for reported runs. Operator coverage is
incomplete and unsupported ops fall back to CPU *silently*, so a latency figure
measures an unknown mixture of devices. CPU is the defensible target on a Mac.

**Timing validity is enforced, not documented.** `assert_timing_reportable()`
raises rather than letting an unusable number reach a table, and
`table_profiling()` adds a Hardware column automatically whenever rows span
devices and prints `n/r` for any timing from an exploration, MPS, hosted-CPU, or
migrated run. On CPU there is no VRAM, so peak RSS is reported and labelled as
such — absent, not zero.

**One project-specific consequence:** the determinism exception below is
CUDA-only. `adaptive_avg_pool2d_backward` has a deterministic CPU kernel, so a
fully deterministic SRNet reference run is possible at `env=local` — slowly.

## Kaggle data acquisition

Off by default (`kaggle.enabled=false`); BSDS500, BOSSBase, DIV2K and COCO are
fetched from their primary sources. It matters because all four are *also*
mirrored on Kaggle, and a mirror is exactly where "the dataset" becomes
ambiguous — mirrors get re-uploaded and gain rows without changing their name.

- **Credentials** are detected at runtime: Colab Secrets on a hosted runtime,
  `~/.kaggle/kaggle.json` at `chmod 600` locally (the client refuses looser).
  `kaggle.json`, `.kaggle/`, `.env` and `*.key` are gitignored before the first
  commit. A key that has ever been committed must be **rotated** — history
  retains the blob, so deleting the file is not sufficient, and
  `ensure_gitignored()` raises if it finds one tracked.
- **A 403 is reported as itself.** Competition data requires accepting the rules
  on the website first, and the API returns a 403 indistinguishable from an auth
  failure until you do. This is the most common false alarm in this path, so it
  is named rather than sending you to debug credentials.
- **Version + SHA-256 are pinned**, compared on every run, and drift is loud but
  never auto-upgraded. A dataset gaining rows between your ablation and your
  final table is a reproducibility failure no seed will catch. A cached archive
  failing its hash is re-downloaded, not used.
- **The leakage trap.** A competition's `test` directory is not a held-out test
  set: it is unlabeled, so no metric can be computed locally, and the leaderboard
  behind it has been probed by thousands of submissions — a validation set with
  extra steps. `assert_not_leaderboard_split()` refuses it. Carve the test split
  from labeled data before any preprocessing and report leaderboard standing
  separately, as context.
- **Pretrained weights are an experimental variable**, not infrastructure:
  `kagglehub` model handles are pinned like datasets. Licenses are recorded;
  many Kaggle datasets forbid redistribution, which constrains what the demo
  notebook may ship.

## Repository layout

```
configs/          Hydra YAML: dataset/, detector/, attack/, embed/, hparams/,
                  env/, supervisor/, ablation/, kaggle.yaml, git.yaml
src/
  data/           cover sourcing, group-aware splits, paired batching, Kaggle pinning
  models/         XuNet, YeNet, SRNet, SRM bank; embedding/ for the baselines
  attacks/        budget bookkeeping, gradient attacks, feature-level attacks
  training/       per-detector recipes, trainer with best-val + auto-resume
  evaluation/     metrics, transferability, significance, plots, tables, profiling
  supervision/    trip conditions, watcher, patch stub, run registry, intervention log
  utils/          determinism, execution target, provenance, git, checkpointing
scripts/          entry points (thin; all logic lives in src/)
                  prepare_data · train_detectors · run_attack · run_embedding_comparison
                  tune (Optuna) · run_supervisor · aggregate_results · smoke_test
tools/            manuscript checker
tests/            one test per reviewer-visible property
notebooks/        demo.ipynb — one-click reviewer reproduction
```

## Things worth knowing before you change anything

**Paired batching is load-bearing.** Steganalysis CNNs learn a residual that is
tiny next to image content. If a batch mixes unrelated covers and stegos, the
gradient is dominated by content and the network settles on a constant
prediction — which is exactly the XuNet failure in the submitted Table 2
(FPR 1.000, FNR 0.000). `PairedBatchSampler` puts each cover and *its own* stego
in the same batch so content cancels. Do not replace it with a shuffled sampler.

**Augmentation must not resample pixels.** Only the D4 group (90° rotations and
flips) is applied, and identically to both members of a pair. Scaling, rotation
by other angles, or JPEG would destroy the embedding residual and train the
detector on nothing.

**The luma projection appears twice and must agree.** `data/pair_dataset.py::_load_gray`
and `attacks/common.py::to_luma` both implement BT.601. If they diverge, the
attack optimises a different signal from the one the detector reads, and the
evasion numbers become meaningless without anything visibly failing.

**Quantization is part of the threat model.** An adversarial image that only
exists in float32 is not a threat; the evader has to write a PNG. Attacks
quantize to the 8-bit grid before evaluation.

**Determinism exceptions.** `torch.use_deterministic_algorithms(True, warn_only=True)`
is used rather than the strict form, because `adaptive_avg_pool2d_backward` —
reached through SRNet's global pooling head — has no deterministic CUDA kernel in
torch 2.4. This is the only known non-deterministic op in the pipeline; it
affects gradients during training, not evaluation, and multi-seed reporting
absorbs the residual variation. Everything else is locked.

## What this code does not claim

It does not claim that the transferability conclusion is true. It makes the
conclusion *checkable*: if SRNet converges under the recipe in
`configs/detector/all.yaml` and the gap remains large across eight seeds and ten
attacks, the claim is supported and `significance.py::claim_supported` will say
so. If SRNet converges and the gap collapses under NAA, the claim was an artefact
of attacking with PGD only, and the paper should say that instead.
