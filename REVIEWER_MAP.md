# Reviewer comment -> code map

Every comment from both reviewers, and the file, function or config that answers
it. Line references are to functions rather than line numbers so they survive
edits.

## Reviewer 1

| # | Comment | Where it is answered |
|---|---|---|
| 1 | Acronyms must be defined before use | `tools/check_manuscript.py::audit_acronyms` — scans the body (not the bibliography) and reports every acronym whose first use has no preceding definition, with the expansion to insert. Run it before resubmission. |
| 2 | "PSNR ≥ 37 dB" contradicts the 36.9 dB in Table 3 | `src/attacks/common.py` separates three quantities that shared one name: `psnr_worst_case` (algebraic floor), `psnr_empirical` (measured), and the manuscript claim. `format_psnr` applies one decimal rounding rule everywhere; `check_psnr_claim` fails the claim against the table at printed precision. Table 3 is emitted by `tables.py::table_evasion_sweep`, whose note states the rule. |
| 3 | Make the threshold statement numerically consistent | Same as above. `tests/test_metrics.py::test_psnr_claim_checker_catches_the_reported_inconsistency` uses the paper's own numbers and asserts the ≥37 dB claim fails while ≥36 dB passes. |
| 4 | XuNet AUC 0.489 vs 0.000 after attack — was polarity fixed? | `src/evaluation/metrics.py::canonical_score` fixes `s = z_stego − z_cover` once; `STEGO_LOGIT_INDEX` and `assert_label_convention` prevent inference from data. `detection_report` returns `inverted` and `auc_flipped` so both appear in the table. `tools/check_manuscript.py::audit_auc_consistency` cross-checks the clean table against the ε=0 row of the sweep. |
| 5 | Transferability interpretation invalid because SRNet starts at 0.498 | `src/evaluation/transferability.py::transferability_gap` returns `defined=False` and `gap=None` when the held-out detector fails the gate, with the reason recorded. Figures draw undefined points as open markers (`plots.py::plot_transfer_gap`). |
| 6 | Committee mean 0.745 includes a member at 0.489 | `metrics.py::committee_summary` returns `mean_auc=None` plus a reason when any member fails `ConvergenceGate`; `tables.py::table_clean_detection` prints the reason in the table note instead of a number. |
| 7 | No comparison with state-of-the-art, especially post-2024 | `configs/attack/suite.yaml` runs ten attacks at a matched budget (through FIA 2021 and NAA 2022); `configs/embed/comparison.yaml` runs eight embedders at matched payload. `tables.py::table_attack_comparison` and `table_embedding_comparison` emit the head-to-head tables with Holm-corrected paired p-values. |
| 8 | Weaken or revalidate the transferability conclusion | The revalidation path: `configs/detector/all.yaml` gives each detector its published recipe (SRNet: Adamax, 450 epochs, step drop, optional payload curriculum); `training/trainer.py` selects on best validation AUC and reports `converged`; `scripts/train_detectors.py` prints the convergence guard and writes `convergence_seed*.json`. Claims are gated by `significance.py::claim_supported`. |

## Reviewer 2

| # | Comment | Where it is answered |
|---|---|---|
| 1 | Evidence preliminary; two of three detectors do not converge | Same convergence work as R1-8. `ConvergenceGate` is applied at training time, at committee assembly, and at gap computation, so a non-converged detector cannot silently enter a result. |
| 2 | No head-to-head comparison with post-2024 transferable attacks, incl. neuron attribution | `src/attacks/feature_level.py` implements FIA (ICCV 2021) and NAA (CVPR 2022) — the neuron-attribution family named in the comment — alongside eight gradient attacks in `src/attacks/gradient.py`. All share `CommitteeLoss`, ε, step size and iteration count. |
| 3 | Held-out SRNet near chance invalidates the transferability claim | `transferability_gap` refuses to report; `configs/detector/all.yaml::srnet` carries the long schedule and curriculum needed to fix it. |
| 4 | Committee is effectively one detector plus two unstable measurements | `committee_summary` suppresses the mean; `data/pair_dataset.py::PairedBatchSampler` addresses the likely cause of XuNet's collapse (unpaired batching). |
| 5 | Insufficient comparison with classical and modern deep hiding under matched settings | `src/models/embedding/classical.py` (LSB replacement/matching, HILL, S-UNIWARD with a payload-constrained ternary simulator) and `src/models/embedding/deep.py` (HiDDeN, retrainable at any payload; SteganoGAN wrapper). `scripts/run_embedding_comparison.py` enforces the matched payload and records the achieved rate. |
| 6 | Reproducibility: seeds, splits, preprocessing, optimizer, schedule, attack loss, ε→PSNR, PGD steps, checkpoint criteria | `utils/determinism.py` (seed locking + provenance), `configs/` (every value), `utils/checkpoint.py` (full-state, RNG-inclusive, auto-resume), `attacks/common.py` (ε→PSNR), `training/trainer.py` (best-val selection), `requirements.txt` (pinned), `notebooks/demo.ipynb` (one-click). |
| 7 | Several numerical claims overinterpreted | `significance.py::claim_supported` fails a claim whose margin is inside the seed-to-seed spread; `tools/check_manuscript.py::audit_hedging` flags strong claim verbs in sentences about transfer, committee or deployment. |
| 8 | Scope too narrow: one dataset, one payload, no uncertainty | `configs/dataset/` (BSDS500, BOSSBase, DIV2K, COCO), `configs/embed/comparison.yaml` (six payloads), `configs/main.yaml::seeds` (eight seeds — see below), `split.salt` for repeated random splits, DeLong CIs on every AUC. |

### Provenance and supervision

Two mechanisms support several answers above rather than any single comment.

**Git provenance** (`src/utils/git_provenance.py`) makes R2-6 checkable rather
than asserted: a commit is made when validation improves, and its message carries
the seed, the resolved-config hash, the W&B run id and the trigger. Asking "what
produced this number" is answered by `git log`, not by reconstruction.

**Autonomous supervision** (`src/supervision/`) is the standing answer to R1-8
and R2-1. The submitted run failed silently — training loss fell for a hundred
epochs while two detectors sat at chance, and nobody noticed until the paper was
written. The supervisor's `underperformance` trigger uses the convergence gate as
its baseline and fires at epoch 20. Its budget is three rounds, persisted in the
checkpoint, and the supervisor cannot see the test split (`assert_no_test_split`,
tested against five naming conventions).

Because that machinery can alter the experiment, it also creates a manuscript
risk the reviewers did not raise: after autonomous edits, the methods section may
no longer describe what ran. `method_altering_diff_summary()` prints a
consolidated diff summary at end of run, and every intervention is logged to
`outputs/agent_interventions.jsonl` and committed. Any method-altering edit that
survives into the final results must be described in the paper.

### Execution target and hardware honesty

Not raised by either reviewer, but load-bearing for R2-6 and R2-8. The submitted
manuscript reports no hardware at all and describes the attack as "low-cost"
without measuring it. The revision declares an execution target per run
(`configs/env/`), pins BLAS threads and records the count next to the seed
(BLAS reduction order changes the floats), and refuses to print a timing that
cannot be compared — exploration threading, Apple MPS, a hosted CPU VM, or a
migrated run all yield `n/r` with the reason in the table note. Where rows span
devices, `table_profiling` adds a Hardware column automatically, because a table
whose baseline ran on a laptop and whose proposed method ran on a T4 is not a
comparison.

### On the number of seeds

`configs/main.yaml` specifies eight seeds rather than five. With five paired
observations the two-sided Wilcoxon signed-rank test cannot produce a p-value
below 0.0625 — its floor is 2^−(n−1) — so a five-seed study using a rank test is
guaranteed a null result regardless of effect size.
`significance.py::min_achievable_p_wilcoxon` detects this and falls back to the
paired t-test with a recorded note, but running eight seeds removes the problem
rather than working around it.
