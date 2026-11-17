# PROMPT.md — Cycle-Count Study (Paper 5) · Claude Code project brief

> **How to use this file.** Put it in an empty folder, open Claude Code there, and send this as the first message:
>
> `Read PROMPT.md completely, then start Phase 0. Stop at every GATE and report before continuing.`
>
> Claude Code should copy the "Standing rules" section into `CLAUDE.md` in Phase 0 so the rules persist across sessions.

---

## 0. What this project is

You (Claude Code) are implementing a complete research study, end to end, for a PhD student in Computing (Multimedia University, Malaysia). The study will become a journal paper.

**Research question.** How does the temporal unit fed to a lung/heart sound classifier — 1, 2, 3 or 4 annotated breath cycles, a fixed-length window, or the whole recording — change diagnostic accuracy, calibration and time-to-decision? Clinicians listen to 2–3 cycles per site; models are usually trained on arbitrary fixed windows; nobody has tested the choice.

**Working title.** *How many cycles does a model need? A controlled study of cycle-based inputs, fixed windows and time-to-decision for lung and heart sound classification.*

**Hypotheses** (the results must answer each with yes/no and a number):

- **H1** Accuracy rises from 1 to 2–3 cycles and plateaus; the gain from the 3rd cycle is smaller than from the 2nd.
- **H2** At equal duration, cycle-aligned inputs beat fixed windows that cut through cycle boundaries (lung and heart).
- **H3** The effect holds across a compact CNN, AST and BEATs (property of the input unit, not of one backbone).
- **H4** With automatic (detected) cycle boundaries, absolute scores fall but the ordering of conditions is unchanged.

**Research questions → analyses**

| RQ | Analysis | Primary outcome |
|---|---|---|
| RQ1 accuracy vs k | one model per k; Score vs k with 95 % CIs over seeds | ICBHI Score; SPRSound event F1; CirCor murmur metrics |
| RQ2 cycle-aligned vs matched fixed window | paired difference per seed at matched duration | ΔScore |
| RQ3 architecture generality | repeat RQ1–RQ2 per backbone; test k × backbone interaction | mixed-model interaction |
| RQ4 time-to-decision | feed cycles one at a time; first cycle after which the answer holds | cycles-to-stable-decision, seconds saved |
| RQ5 automatic boundaries | re-score with detector-produced boundaries | ΔScore and rank-order preservation |
| RQ6 (optional) noise | re-run RQ1 at 10 dB and 0 dB SNR with public siren/road noise | Score vs k per SNR |

**Nearest prior work you must position against (cite, reproduce the relevant idea, do not copy code):**
PC-MCL (arXiv 2601.17080, Jan 2026 — concatenates ICBHI cycles into a fixed 10 s input, 3-label formulation, Score 65.37); Bao, Xu & Kamavuako, *Sensors* 2022 (heart-sound duration 1–5 s); Patch-Mix CL, Interspeech 2023 (the ICBHI baseline codebase).

---

## 1. Environment and division of labour

- **Local machine (where you run):** the git repository, all code, configs, manifests, statistics, figures, paper. CPU only — never train here beyond smoke tests.
- **Google Colab via the Colab MCP server:** every download, feature extraction, training run and GPU evaluation. **Before anything else, list the Colab MCP tools you actually have** (execute cell / read output / upload / download / list files, etc.) and write down in `docs/COLAB_WORKFLOW.md` the exact loop you will use. Do not assume capabilities you have not verified.
- **Google Drive (mounted in Colab) is the only persistent storage.** Colab VMs are ephemeral. Everything that takes more than 10 minutes to recreate — raw datasets, cached features, checkpoints, logs, `results.csv` — lives under `/content/drive/MyDrive/cycle-count-study/`. At the start of every Colab session: mount Drive, `git clone`/`git pull` the repo, `pip install -r requirements.txt`, and verify the dataset checksums.
- **Code flows one way:** written locally → committed → pushed to the GitHub remote (ask the user for the remote URL in Phase 0; if none exists, create a private repo with `gh repo create`) → pulled in Colab. Results flow back: Colab writes `results/results.csv`, logs and figures to Drive, you download them (or `git commit` them from Colab) into the local repo.
- **Every run must be resumable and idempotent:** before starting a run, check whether its row already exists in `results.csv` (keyed by dataset, condition, k, window_s, backbone, boundary_source, seed, config_hash) and skip if so. Checkpoint every epoch to Drive. A Colab disconnect must cost at most one epoch.
- **GPU budget is unknown until you ask.** In Phase 0 ask the user which Colab tier/GPU they have (free T4, Pro L4/A100). Then compute a run-time budget table (Section 7) and get it approved at GATE 1 before launching any grid.

---

## 2. Datasets (links supplied by the user — use exactly these)

| Dataset | Role | Link / method | Notes |
|---|---|---|---|
| ICBHI 2017 Respiratory Sound Database | **primary lung benchmark** | `kagglehub.dataset_download("vbookshelf/respiratory-sound-database")` | 920 recordings, 126 subjects, 6,898 annotated cycles with start/end + crackle/wheeze flags; official 60/40 patient-wise split file (`ICBHI_challenge_train_test.txt`) is inside the Kaggle package — use it. Sample rates 4–44.1 kHz. If kagglehub asks for credentials, ask the user for their Kaggle API token; never guess. |
| SPRSound (SJTU paediatric) | lung replication | `git clone https://github.com/SJTU-YONGFU-RESEARCH-GRP/SPRSound.git` | Use the **2022** split (1,949 train / 734 test) for training/testing and the **2023** test set (871 files) as an extra held-out set. Event JSONs carry start/end (ms) and type: Normal, Rhonchi, Wheeze, Stridor, Coarse Crackle, Fine Crackle, Wheeze+Crackle. 8 kHz. |
| CirCor DigiScope PCG 1.0.3 | **primary heart benchmark** | `https://physionet.org/content/circor-heart-sound/get-zip/1.0.3/` | Public release: 3,163 recordings, 942 subjects, 4 kHz, 5–65 s. Per-recording site (AV/PV/TV/MV/Phc) and murmur label; `.tsv` segmentation files give S1/systole/S2/diastole state intervals → cardiac cycles = S1 onset to next S1 onset. Also patient-level murmur (Present/Absent/Unknown) and outcome labels. |
| PhysioNet/CinC 2016 | heart replication (sanity) | `https://physionet.org/content/challenge-2016/get-zip/1.0.0/` | 2 kHz, normal/abnormal per recording, 5–120 s. Use Springer's LR-HSMM for S1 positions (Section 6.4). Only the two best heart conditions are replicated here. |
| HF_Lung_V1 | training data for the **breath-phase detector** (RQ5) | `git clone https://gitlab.com/techsupportHF/HF_Lung_V1_IP.git` | 9,765 × 15 s recordings; inhalation (34,095) and exhalation (18,349) interval labels + CAS/DAS labels. Used only to train the detector; never mixed into classification training. |

Rules: raw downloads go to `drive/.../data/raw/<dataset>/` and are **never modified**; write a `data/CHECKSUMS.md` with file counts and SHA-256 of archives; respect each licence (CirCor ODC-By; SPRSound CC BY; cite all five). If any download fails or looks incomplete (file counts off), **stop and ask** — do not substitute another copy.

---

## 3. Repository layout (create exactly this)

```
cycle-count-study/
  CLAUDE.md                # standing rules (Section 9), copied in Phase 0
  PROMPT.md                # this file
  README.md
  requirements.txt         # pinned
  docs/
    COLAB_WORKFLOW.md      # the verified Colab loop
    DECISIONS.md           # every design decision + date + reason (append-only)
    RESULTS.md             # living results summary, updated at each gate
  data/                    # local: only CHECKSUMS.md and tiny fixtures (raw data lives on Drive)
  manifests/               # one CSV per dataset × boundary_source — the heart of the study
  configs/                 # one YAML per run family; condition chosen by config
  src/cyclestudy/
    data/                  # loaders, resampling, annotation parsers (ICBHI txt, SPRSound json, CirCor tsv)
    units/                 # k-cycle, fixed-window, whole-recording builders → manifest rows
    features/              # log-mel front-end with caching to Drive
    detect/                # HF_Lung breath-phase detector; HSMM wrapper for heart
    models/                # compact CNN, AST wrapper, BEATs wrapper
    train/                 # single entry point `train.py --config ...`
    eval/                  # ICBHI Score, F1, AUROC, ECE, Brier; sequential-decision analysis
    stats/                 # bootstrap, mixed model, TOST, Holm; figure scripts
  tests/                   # pytest: builders, label logic, split assertions, metric definitions
  results/
    results.csv            # one row per run (schema in Section 6.7)
    figures/
    tables/
  paper/                   # LaTeX skeleton, figures, supplementary manifest description
```

Python 3.10+, PyTorch ≥ 2.1, torchaudio, numpy, pandas, scikit-learn, statsmodels, scipy, matplotlib, pyyaml, tqdm, pytest. Third-party model code: AST from `YuanGongND/ast` (or `transformers` AST), BEATs from `microsoft/unilm/beats`; the Patch-Mix codebase `raymin0223/patch-mix_contrastive_learning` as a **git submodule used for baseline reproduction and as a reference for the ICBHI split/metric/augmentation** — do not fork its training loop until the baseline reproduces.

---

## 4. Phases and GATES

At every **GATE**: commit, update `docs/RESULTS.md`, print a short report (what was done, numbers, open questions, what you will do next), and **wait for the user to say "continue"**.

### Phase 0 — Setup (no GPU)
1. Ask the user: GitHub remote URL (or permission to create one), Colab tier/GPU, Kaggle token availability, whether to run RQ6.
2. Create the repo layout, `CLAUDE.md`, `requirements.txt`, `README.md`, `docs/DECISIONS.md`.
3. Verify the Colab MCP tools; write `docs/COLAB_WORKFLOW.md`; prove the loop with a cell that mounts Drive, prints GPU name, and clones the repo.
4. Download all five datasets to Drive; write `data/CHECKSUMS.md`; print per-dataset file counts and compare with the expected counts above.
5. **GATE 0.**

### Phase 1 — Baseline reproduction (GPU)
1. Add the Patch-Mix submodule; run its AST recipe on the **official** ICBHI 60/40 split with its defaults.
2. Expected: ICBHI Score ≈ 62–64. If below 60, debug (sample rate, padding, split file) before proceeding. If it cannot be reproduced, stop and report — nothing downstream is comparable without it.
3. Record the exact command, commit hash, GPU, wall-clock per epoch, and Score in `docs/RESULTS.md`.
4. **GATE 1** — also present the compute budget table (Section 7) for approval.

### Phase 2 — Manifests and unit builders (CPU)
1. Parse annotations for ICBHI, SPRSound, CirCor, PhysioNet 2016.
2. Implement the three builders (Section 6.1) and write `manifests/icbhi_annotated.csv`, `manifests/sprsound_annotated.csv`, `manifests/circor_annotated.csv`.
3. Implement and run the assertions (Section 6.2) as pytest tests. All must pass.
4. Produce `results/tables/manifest_summary.csv`: units per condition, duration distribution (median, IQR), label distribution — this becomes Table 1.
5. Feature caching: precompute log-mel for every unit on Colab, store as `.npy`/`.pt` shards on Drive keyed by unit hash.
6. **GATE 2.**

### Phase 3 — Compact-CNN grid on ICBHI (GPU, fast)
1. Run all 11 conditions × 5 seeds with the compact CNN. Expect hours, not days.
2. Add the **unit-count control**: k = 1 subsampled to the unit count of k = 3, 5 seeds.
3. Produce the first Score-vs-k figure. If the curve is flat or noisy beyond CIs, inspect builders/labels before any transformer run.
4. **GATE 3.**

### Phase 4 — Transformer grids (GPU, long; run in background, poll)
1. AST grid on ICBHI (conditions and seeds per the approved budget).
2. BEATs grid (reduced set if budget-limited: k = 1–4 + matched windows + 8 s).
3. Meanwhile (CPU): CirCor and SPRSound manifests and feature caches; OSF pre-registration text in `docs/PREREG.md` (hypotheses, primary endpoint, factor grid, analysis plan) for the user to submit.
4. **GATE 4.**

### Phase 5 — Heart and paediatric replications (GPU)
1. CirCor: k = 1, 3, 5, 10 and matched windows + 3 s and 5 s; CNN and AST; seeds per budget.
2. SPRSound: k-cycle vs matched window; CNN and AST.
3. PhysioNet 2016: best two heart conditions only.
4. **GATE 5.**

### Phase 6 — Detector and RQ5 (GPU small)
1. Train the breath-phase detector on HF_Lung_V1 (Section 6.4); validate on ICBHI annotated boundaries (onset error ms, missed/split cycles).
2. Wrap Springer LR-HSMM (or train a small S1/S2 frame model on CirCor `.tsv`) for heart.
3. Rebuild **test** manifests with detected boundaries (`*_detected.csv`); re-score every k-cycle model; also run fully-automatic k = 3 (detected boundaries in training too) as a stress test.
4. **GATE 6.**

### Phase 7 — Time-to-decision (CPU/GPU small)
Implement Section 6.6; produce the accuracy-vs-cycles figure (two rules) and seconds-saved panel; per-class stopping-cycle table. **GATE 7.**

### Phase 8 — Statistics and figures (CPU)
Freeze `results.csv`; run Section 6.8; generate every figure/table from scripts into `results/figures` and `results/tables`; write `docs/RESULTS.md` with a yes/no + number for H1–H4. **GATE 8.**

### Phase 9 — Paper (CPU)
LaTeX skeleton in `paper/` with sections, figure/table placeholders wired to the generated files, methods text drafted from the actual configs and manifests (numbers pulled from `results.csv`, never typed by hand), reproducibility statement, data availability. **GATE 9 — final.**

---

## 5. Non-negotiable scientific rules

1. **Official patient-wise splits only.** ICBHI: the challenge train/test file. SPRSound: the 2022 train/test (plus 2023 test). CirCor: stratified patient-wise split (fixed seed, saved to `manifests/circor_split.csv`). No patient may appear in two splits — a test enforces it.
2. **No selection on the test set.** Carve a patient-wise validation fold (20 % of training patients, label-stratified) from the training split; select epochs and temperature-scale on it; touch the test set once per run.
3. **Identical recipe across conditions.** Features, augmentation, optimiser, schedule, early-stopping rule and class weighting are fixed once (tuned on k = 1 only) and never tuned per condition. Log the config hash in every results row.
4. **Same test recordings in every condition.** A recording that yields no unit under some condition (too short) is dropped from the test set of **all** conditions of that dataset — enforced by a test.
5. **Never fabricate or hand-type a number.** Every number in `docs/RESULTS.md` and `paper/` comes from `results.csv` or a script. If a run is missing, say so.
6. **Raw data is read-only.** Derived data lives in cache directories keyed by content hash.
7. **Every result row carries:** dataset, condition, k, window_s, boundary_source, backbone, seed, config_hash, commit, epoch_selected, Sp, Se, Score, F1_macro, per-class F1, AUROC (binary tasks), ECE, Brier, n_train_units, n_test_units, train_minutes, params, GPU name.

---

## 6. Implementation specifications

### 6.1 Unit builders (lung: ICBHI, SPRSound)
- Resample to **16 kHz mono**. Keep original sample rate in the manifest.
- **k-cycle units:** k consecutive annotated cycles, stride one cycle; keep a unit only if all cycles are complete. A 7-cycle recording yields 7/6/5/4 units for k = 1/2/3/4.
- **Labels for a multi-cycle unit:** store per-cycle labels; derive (a) the 4-class ICBHI label = {normal, crackle, wheeze, both} from "crackle in any cycle / wheeze in any cycle", and (b) the 3-label multi-label formulation {normal, crackle, wheeze} as in PC-MCL. Report both; (a) is primary for Score.
- **Length handling:** model input 8 s for k ≤ 3, 12 s for k = 4; shorter units are **cyclically repeated** (not zero-padded); longer units are centre-cropped and `cropped=True` logged.
- **Fixed-window controls:** for each k, windows whose length = median duration of k-cycle units in the **training** set (expect ≈ 2.5 / 5 / 7.5 / 10 s), random start offsets (seeded), labels from any cycle overlapping the window by > 50 %. Plus the two literature conventions: 8 s and 10 s.
- **Whole recording:** full recording, recording-level label (any crackle / any wheeze).
- SPRSound: "cycles" are the annotated respiratory events; map classes to {normal, crackle (fine+coarse), wheeze (wheeze+rhonchi+stridor), both} for the ICBHI-style task and also keep the native 7-class event task.

### 6.2 Assertions (pytest, must pass before any training)
- No `patient_id` in more than one split, for every manifest.
- For each dataset, the set of test `recording_id`s is identical across all conditions.
- Every unit's `[start, end]` lies inside its recording; k-cycle units contain exactly k complete cycles.
- Label derivation round-trips: 4-class label == function(per-cycle flags).
- Metric unit tests: ICBHI Score on a hand-built confusion matrix equals the known value.

### 6.3 Heart units (CirCor, PhysioNet 2016)
- Native 4 kHz / 2 kHz; upsample to 16 kHz only inside the feature front-end for pretrained backbones.
- Cardiac cycle = S1 onset to next S1 onset from the `.tsv` states (CirCor) or LR-HSMM output (2016). k ∈ {1, 3, 5, 10}, stride 1. Fixed-window controls at median duration of k cycles, plus 3 s and 5 s.
- Labels per recording (site-level murmur Present/Absent; drop Unknown for the main task). Evaluate at unit level and at recording level (mean posterior). Report Present-vs-Absent F1 and AUROC; additionally the PhysioNet 2022 weighted accuracy on patient-level 3-class murmur labels for comparability.

### 6.4 Breath-phase detector and heart segmenter (RQ5)
- Detector: CNN-BiGRU frame classifier {inhalation, exhalation, other} on HF_Lung_V1 log-mel frames (10 ms hop); post-process: hysteresis on posteriors, minimum cycle length 1.0 s; cycle boundary = inhalation onset. Validate on ICBHI boundaries: median onset error (ms), % cycles missed, % cycles split. Report these in Table 4.
- Heart: Springer LR-HSMM (PhysioNet `hss` code; a Python port is acceptable if its output is validated against CirCor `.tsv` S1 positions on a held-out subset) → S1 onsets.
- Training manifests keep annotated boundaries; **test** manifests are rebuilt with detected boundaries. Also run the fully automatic k = 3 variant.

### 6.5 Models and recipe
- **Compact CNN:** 4 conv blocks (16/32/64/128 filters, 3×3, BN, ReLU, max-pool), dropout 0.3, global average pooling, 3 dense layers, softmax. Must be < 1 M parameters. Train from scratch, AdamW lr 1e-3, 100 epochs, batch 32, cosine decay.
- **AST:** ImageNet+AudioSet pretrained; AdamW lr 5e-5, 50 epochs, batch 8, cosine decay, mixed precision.
- **BEATs:** AudioSet self-supervised checkpoint (iter3+); same recipe as AST.
- **Front-end:** 16 kHz, 128-bin log-mel, 25 ms window, 10 ms hop, per-dataset mean/var normalisation from the training fold only.
- **Augmentation (identical everywhere):** SpecAugment (2 freq masks ≤ 24 bins, 2 time masks ≤ 10 % length) + the Patch-Mix policy from the codebase. No noise injection in the main grid.
- **Imbalance:** class-weighted cross-entropy (inverse frequency, square-root dampened). No oversampling.
- **Seeds:** 5 for the CNN; transformers 5 if budget allows, else 3 (state in paper). Seeds 0–4, deterministic data order per seed.
- **Model selection:** best validation Score epoch; temperature scaling fitted on validation.

### 6.6 Time-to-decision (RQ4)
- For each ICBHI test recording, present cycles cumulatively: after cycle 1 use the k = 1 model, after 2 the k = 2 model, … up to k = 4, then slide the k = 4 model.
- Stopping rules: (a) **stable answer** — first cycle after which the predicted class never changes for the remainder; (b) **confident answer** — first cycle at which max posterior ≥ τ, τ ∈ {0.6, 0.7, 0.8, 0.9, 0.95} (temperature-scaled posteriors).
- Report: distribution of stopping cycle (median, IQR) overall and per class; accuracy of the decision taken at the stopping cycle vs ground truth and vs the full-recording decision; seconds saved vs a 10 s fixed window and vs the whole recording.
- Repeat for CirCor with cardiac cycles (expect 5–10 cycles).

### 6.7 Results schema
`results/results.csv` columns exactly as listed in Section 5.7, plus `timestamp` and `notes`. Append-only; a run is skipped if its key already exists. A second file `results/sequential.csv` holds the RQ4 outputs (recording_id, rule, tau, stopping_cycle, decision, truth, seconds).

### 6.8 Statistics
- Per condition: mean ± 95 % CI over seeds **and** patient-level bootstrap (1,000 resamples of test patients).
- RQ1/RQ3: linear mixed model (`statsmodels` MixedLM), Score ~ k (ordered) × backbone, random intercept per seed; contrasts 2 vs 1, 3 vs 2, 4 vs 3.
- RQ2: paired differences (cycle-aligned − matched window) within seed; report difference with CI and sign consistency across backbones/datasets.
- Equivalence: TOST on k = 3 vs k = 4 with ±2 Score-point bounds.
- Holm correction within each RQ; report effect sizes (raw ΔScore with CI, Cliff's delta). Never a p-value without a magnitude.

---

## 7. Compute budget (fill in at GATE 1)

Build `docs/BUDGET.md` with measured minutes/epoch per backbone on the available GPU, then the total for each grid, then a proposal:

- **Tier A (must):** CNN full grid (11 conditions × 5 seeds) + unit-count control; AST on k = 1–4, matched windows, 8 s (3–5 seeds).
- **Tier B (should):** BEATs on k = 1–4 + 8 s; CirCor CNN + AST; SPRSound AST.
- **Tier C (nice):** full BEATs grid; PhysioNet 2016; RQ6 noise sweep.

Techniques to cut cost without changing the science: cache features on Drive once; mixed precision; early stopping with patience 10 on the validation fold; run transformer grids as background processes in Colab writing logs to Drive, and poll rather than block; split grids into batches that fit one Colab session; always resume from checkpoints.

---

## 8. Reporting conventions

- `docs/RESULTS.md` is rewritten at every gate: a table per RQ, the current answer to each hypothesis, and open issues.
- Every figure is produced by a script in `src/cyclestudy/stats/` and saved as PDF + PNG with the generating command in its caption file.
- `docs/DECISIONS.md` is append-only: date, decision, reason, alternatives rejected.
- When something cannot be done (download blocked, model weights unavailable, Colab limit), say so plainly and propose the smallest change; do not silently substitute.

---

## 9. Standing rules (copy into CLAUDE.md)

- Read `PROMPT.md` and `docs/DECISIONS.md` at the start of every session; check `results/results.csv` to see what is already done.
- Never train on the local machine; never store data in the repo; never edit raw data.
- Official patient-wise splits; validation fold for all selection; identical recipe across conditions; same test recordings across conditions.
- Every number reported comes from `results.csv` or a script; cite the file and row.
- Stop at every GATE; stop when a download or weight fetch fails; stop before any run that will exceed the approved budget.
- Commit after every completed task with a message that names the phase and task; push before starting Colab work.
- Keep `docs/RESULTS.md`, `docs/BUDGET.md`, `docs/COLAB_WORKFLOW.md` current.
- Write tests for builders, labels and metrics before writing training code.
