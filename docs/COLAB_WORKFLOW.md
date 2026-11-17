# COLAB_WORKFLOW.md — the verified Colab loop

Verified 2026-11-17 against a live Colab session (Python 3.13.18).

## Available MCP tools (verified, server `colab-proxy-mcp`)

| Tool | What it does |
|---|---|
| `open_colab_browser_connection` | attaches to the Colab tab open in the user's browser; must succeed before any other tool works |
| `get_cells` | read all cells (optionally with outputs) |
| `add_code_cell` / `add_text_cell` | insert a cell at an index; returns cell id |
| `update_cell` | overwrite a cell's content by id |
| `run_code_cell` | execute a cell **synchronously**; returns its outputs |
| `delete_cell`, `move_cell` | housekeeping |

**There is no file upload/download tool.** All file movement goes through git
(code: GitHub → `git clone`/`pull` in a cell) and Google Drive (data/results:
written by cells to `/content/drive/MyDrive/cycle-count-study/`, read back
locally only via small `print()`s or by committing result CSVs from Colab).

## Known constraints (observed)

- The browser connection drops easily (tab refresh, runtime change, browser switch).
  Recovery: user re-opens the Colab tab → `open_colab_browser_connection` → re-`get_cells`
  (cell ids survive if the notebook is the same).
- `run_code_cell` blocks until the cell finishes. **Anything > ~2 min must run as a
  background process** (`subprocess.Popen` / `nohup ... &`) writing a log to Drive,
  with a separate tiny poll cell that tails the log and checks a sentinel file.
- The Colab VM's mounted Drive account is **not** the local machine's Drive account
  (`manipips@gmail.com` locally; the Colab mount has no `PHD/` folder). Repo therefore
  flows via GitHub only. Dataset storage quota is the Colab account's Drive quota —
  check `shutil.disk_usage` / Drive quota before large downloads.
- Current runtime at verification time was **CPU** (fine for Phase 0 downloads);
  Phases 1+ need the user to switch the runtime to an A100 (Pro+) and reconnect.

## The session-start ritual (every Colab session)

One cell, run first:

```python
import os, subprocess, shutil
from google.colab import drive
if not os.path.ismount('/content/drive'):
    drive.mount('/content/drive')
ROOT = '/content/drive/MyDrive/cycle-count-study'
os.makedirs(ROOT, exist_ok=True)
if os.path.isdir('/content/cycle-count-study/.git'):
    subprocess.run(['git','-C','/content/cycle-count-study','pull'], check=True)
else:
    subprocess.run(['git','clone',
      'https://github.com/mabdullahejaz2026/cycle-count-study.git',
      '/content/cycle-count-study'], check=True)
subprocess.run(['pip','install','-q','-r','/content/cycle-count-study/requirements.txt'])
print(subprocess.run(['nvidia-smi','--query-gpu=name,memory.total','--format=csv,noheader'],
      capture_output=True, text=True).stdout if shutil.which('nvidia-smi') else 'CPU runtime')
# then: verify dataset checksums against data/CHECKSUMS.md before any run
```

## Long-running jobs (training, downloads)

1. Launch cell: `subprocess.Popen(['python','train.py','--config',...])` with
   `stdout`/`stderr` redirected to `{ROOT}/logs/<job>.log`; write `<job>.DONE` on exit.
2. Poll cell (re-run cheaply): print `tail -n 20` of the log + check for `.DONE`.
3. Checkpoints go to Drive every epoch; every job is idempotent (skips if its
   `results.csv` row exists) so a VM death costs at most one epoch.

## Results flow back

Colab cells append to `{ROOT}/results/results.csv` on Drive. At each gate, a cell
copies the current `results.csv` + figures into the cloned repo, commits and pushes
(public repo; push uses a token only if/when needed — for now results are synced by
printing/committing from the local side after download via small cells).
