# DECISIONS.md — append-only design decision log

Format: date · decision · reason · alternatives rejected.

---

**2026-11-17 · Create new private GitHub repo `cycle-count-study`.**
Reason: no existing remote; user approved creation via `gh repo create`.
Rejected: existing remote (none available).

**2026-11-17 · Compute budget planned around Colab Pro+ (A100 40 GB).**
Reason: user's subscription tier. Budget table to be measured and approved at GATE 1.
Rejected: assuming free-tier T4 limits (would over-batch the grids unnecessarily).

**2026-11-17 · ICBHI download deferred until the user supplies a Kaggle API token.**
Reason: kagglehub requires credentials; standing rule — never guess credentials.
The other four datasets (SPRSound, CirCor, PhysioNet 2016, HF_Lung_V1) need no token
and download first.

**2026-11-17 · RQ6 (noise sweep) deferred to GATE 1 budget decision (Tier C).**
Reason: user chose to decide once minutes/epoch on the A100 are measured.
Rejected: committing now (no cost data yet) or dropping it outright.

**2026-11-17 · requirements.txt pins exact versions for analysis libraries but floors
(`>=2.1`) for torch/torchaudio.**
Reason: Colab ships its own CUDA-matched torch build; pinning an exact torch version
would force a reinstall mismatched to the VM's CUDA driver. Exact versions recorded per
run in results rows via config_hash + commit.
Rejected: exact torch pin (fragile across Colab image updates).

**2026-11-17 · Repo made public (user decision) so Colab can clone without credentials.**
Reason: the Colab VM mounts a different Google account's Drive, so the repo cannot
flow via Drive; the user preferred public visibility over managing a PAT in Colab
Secrets. Consequence: paper drafts and prereg text will be publicly visible —
revisit before Phase 4 (prereg) if needed.
Rejected: fine-grained PAT in Colab Secrets (more setup); account switch (Pro+
subscription tied to the Colab account).

**2026-11-17 · All Colab work goes through notebook cells; long jobs run as
background processes with log files + DONE sentinels on Drive, polled by small cells.**
Reason: the MCP `run_code_cell` tool is synchronous and the browser connection is
fragile (verified); blocking on a long cell would lose the session.

**2026-11-17 · HF_Lung link discrepancy: PROMPT.md points to `HF_Lung_V1_IP` (the
39-subject Incremental Package, 7z archives) but describes HF_Lung_V1 (9,765 × 15 s
recordings, 34,095/18,349 phase labels).**
Action: kept the IP download untouched and additionally cloned
`gitlab.com/techsupportHF/HF_Lung_V1` (the dataset actually described). To be
confirmed by the user at GATE 0; detector training (Phase 6) will use V1.
