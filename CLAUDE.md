# CLAUDE.md — Cycle-Count Study standing rules

Copied from PROMPT.md Section 9 in Phase 0. These rules persist across sessions.

- Read `PROMPT.md` and `docs/DECISIONS.md` at the start of every session; check `results/results.csv` to see what is already done.
- Never train on the local machine; never store data in the repo; never edit raw data.
- Official patient-wise splits; validation fold for all selection; identical recipe across conditions; same test recordings across conditions.
- Every number reported comes from `results.csv` or a script; cite the file and row.
- Stop at every GATE; stop when a download or weight fetch fails; stop before any run that will exceed the approved budget.
- Commit after every completed task with a message that names the phase and task; push before starting Colab work.
- Keep `docs/RESULTS.md`, `docs/BUDGET.md`, `docs/COLAB_WORKFLOW.md` current.
- Write tests for builders, labels and metrics before writing training code.

## Session decisions (Phase 0, 2026-11-17)

- GitHub: private repo `cycle-count-study` created via `gh repo create`.
- Colab tier: **Pro+ (A100)**.
- Kaggle token: **not yet available** — ICBHI download is blocked until the user supplies `kaggle.json`; never guess credentials.
- RQ6 (noise sweep): deferred to GATE 1 budget decision (Tier C).
- Persistent storage: `/content/drive/MyDrive/cycle-count-study/` on Google Drive (mounted in Colab).
