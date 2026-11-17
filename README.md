# cycle-count-study

**How many cycles does a model need?** A controlled study of cycle-based inputs, fixed
windows and time-to-decision for lung and heart sound classification.

PhD study (Paper 5), Multimedia University, Malaysia. The full project brief, phases and
scientific rules are in [`PROMPT.md`](PROMPT.md); standing rules in [`CLAUDE.md`](CLAUDE.md).

## Research question

How does the temporal unit fed to a lung/heart sound classifier — 1, 2, 3 or 4 annotated
breath cycles, a fixed-length window, or the whole recording — change diagnostic accuracy,
calibration and time-to-decision?

## Layout

- `src/cyclestudy/` — loaders, unit builders, features, detectors, models, training, eval, stats
- `manifests/` — one CSV per dataset × boundary source (the heart of the study)
- `configs/` — one YAML per run family
- `results/results.csv` — one append-only row per run
- `docs/` — DECISIONS.md (append-only), RESULTS.md (per-gate), COLAB_WORKFLOW.md, BUDGET.md
- `paper/` — LaTeX

## Environment

Code is written locally (CPU only) and pushed to GitHub; all downloads, feature extraction
and training run on Google Colab (Pro+, A100) with Google Drive
(`/content/drive/MyDrive/cycle-count-study/`) as the only persistent storage.

## Datasets

ICBHI 2017 (primary lung), SPRSound (paediatric lung), CirCor DigiScope (primary heart),
PhysioNet/CinC 2016 (heart sanity), HF_Lung_V1 (breath-phase detector training only).
Raw data lives on Drive, never in this repo; see `data/CHECKSUMS.md`.
