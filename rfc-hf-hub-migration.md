# RFC: Migrate Lab Data Versioning to Hugging Face Hub (Xet Storage)

| Field | Value |
| --- | --- |
| **Status** | Withdrawn |
| **Author** | Shantanu Singh |
| **Created** | 2026-02-15 |
| **Last updated** | 2026-02-15 |

## Why This RFC Was Withdrawn

After prototyping against `jump_production` and reviewing the design against
the then-current `workflows.md` (the pipeline SOP since superseded by
the [catalog-skills](https://github.com/carpenter-singh-lab/catalog-skills)
vignette-catalog method), we concluded that the complexity is not justified
by the problems it solves — at least not yet. The core issue: **not everything
needs versioning, and S3's simplicity is partly the point.**

The current workflow stores final results on S3 and relies on pipeline
reproducibility (Snakemake + pinned environments) to regenerate anything
upstream. That's a deliberate trade-off — less safety, much less ceremony.
HF Hub would version everything (raw, interim, processed), but the lab hasn't
established that it *needs* all of that versioned. Introducing full data
versioning before answering "what actually needs to be versioned?" adds
complexity without a clear payoff.

### Specific findings

**High severity**:

1. **Exact-sync regression.** `hf download` is additive (adds/updates files,
   never deletes). The current workflow uses `rclone sync`, which converges
   local state to match the remote exactly. The RFC's `get-results-clean`
   recipe papers over this with `rm -rf` + re-download, but only for results —
   no equivalent exists for inputs. This is a step backward in reliability.

2. **Input download scope expansion.** The current daily workflow intentionally
   pulls only `external/` and `profiles/` — a narrow, fast sync. The RFC's
   `get-inputs` pulls all of `raw/**`, which for large projects (Cell Painting
   images) is a major bandwidth and disk regression.

**Medium severity**:

1. **Daily workflow drops `get-inputs`.** The RFC's start-of-day omits input
   sync (`git pull` → `get-results` → `dry`), while the current workflow
   includes it. Analysts can silently drift from upstream inputs.

2. **`pr-create` is misleading.** The recipe says "Creating PR from branch" but
   uploads local `./data` with `--create-pr` and no `--revision`. It doesn't
   actually represent the HF branch state — it uploads whatever is on disk.
   This disconnect between the branching model narrative and the actual command
   will confuse reviewers.

3. **Branch lifecycle is manual and fragile.** Every `git checkout -b` requires
   a matching `just branch-create`. Forgetting this is a known failure mode
   (acknowledged in the Trade-offs section). The current workflow has no
   equivalent extra step. A `post-checkout` hook could automate this, but adds
   yet another hook to maintain.

4. **Role guardrails are weaker.** The current workflow constrains analysts to
   `processed/` and explicitly says "never edit upstream directories." The RFC's
   default `put-results` uploads both `interim/` and `processed/` without
   restating those boundaries.

### Fundamental tension

The RFC tries to version everything because HF Hub makes versioning easy. But
the lab's actual need is narrower: version *final results* and make them
browsable, while keeping inputs reproducible via pipeline re-execution. S3 with
`rclone sync` already handles the "dumb pipe" role well. The missing pieces
(audit trail, conflict prevention, browsability) might be better addressed
individually rather than by replacing the entire data layer.

### What would need to be true to revisit

- A clear answer to "what data layers need versioning?" (probably just
  `processed/`, not `raw/` or `interim/`)
- HF Hub or an alternative that supports exact-sync semantics (delete tracking)
- A branching model that doesn't require manual synchronization between GitHub
  and the data remote
- Cost justification beyond "it's only $200/month" — what incidents would this
  have prevented?

### What was learned

- DVC is not the answer either (daily-use friction, no browsing, file-level
  dedup only, pipeline overlap with Snakemake — see FAQ below)
- HF Hub's chunk-level dedup (Xet) is genuinely valuable for large parquet
  files with incremental changes
- The Justfile pattern for wrapping data operations works well regardless of
  backend
- Web-browsable data (parquet viewer, image preview) is a real quality-of-life
  improvement that S3 does not offer

______________________________________________________________________

## Summary

Replace rclone/s5cmd + S3 with Hugging Face Hub (Xet-backed) for all lab data
synchronization. GitHub remains the home for code. HF Hub becomes the home for
data. Everything else (Snakemake, Cookiecutter structure, pixi, pre-commit,
Justfile) stays the same.

## Motivation

The current S3-based workflow has no versioning, no branching, and no conflict
prevention. Specific problems:

| Problem | Current behavior |
| --- | --- |
| Two people push simultaneously | Last write wins, silent data loss |
| Pipeline update breaks data | Hope someone has a local copy |
| "Which data made this figure?" | Check Slack / guess |
| Review teammate's results | "Push to S3 and I'll pull" |
| Re-upload 5GB file with 1 column changed | Upload entire 5GB again |
| Audit trail | None |

## Proposal

Use HF Hub dataset repos as the data remote. Each repo is a Xet-backed Git
repository with chunk-level deduplication, branching, commit history, and pull
requests — no infrastructure to run.

### What changes

| Aspect | Before | After |
| --- | --- | --- |
| Data remote | S3 bucket directly | HF Hub (backed by S3 + Xet) |
| Sync tool | rclone + s5cmd | `hf` CLI |
| Auth | AWS credentials | HF token |
| Conflict prevention | Slack coordination | Branches + PRs |
| Cost | AWS S3 charges | HF Team plan ($20/user/mo) + excess storage |
| Infrastructure to maintain | None | None (HF is managed) |

### What does not change

- Snakemake pipelines
- Cookiecutter directory structure
- pixi environments
- Notebook naming conventions
- GitHub for code
- `.gitignore` patterns
- `src` layout and Python packages
- Pre-commit hooks

______________________________________________________________________

## Detailed Design

### 1. Hugging Face Organization

The lab org is live: <https://huggingface.co/carpenter-singh-lab>

Ask a maintainer to add you as a member.

### 2. Install tools

```bash
# Add to your pixi dependencies (--pypi required — hf_xet is not on conda-forge)
pixi add --pypi huggingface_hub hf_xet

# Login (each team member, one-time)
pixi run hf auth login
```

The CLI binary is `hf` (not `huggingface-cli`). Since it's a project
dependency, always run it via `pixi run hf` or from within `pixi shell`.

### 3. Project `.env` file

Each project needs a `.env` at the repo root (already in `.gitignore`):

```bash
HF_PROJECT=myproject
```

### 4. Create a repo for your project

Each project gets one dataset repo on HF Hub:

```bash
pixi run hf repo create carpenter-singh-lab/myproject \
  --repo-type dataset --private
```

The repo mirrors the local `data/` directory structure (`external/`, `raw/`,
`interim/`, `processed/`). Use `--include` patterns in download/upload commands
to sync only the layers you need.

For very large projects (millions of images), you may need to split across
repos to stay under the 100k file limit.

### 5. Initial data upload

```bash
# Upload all data layers
pixi run hf upload carpenter-singh-lab/myproject ./data/external external/ \
  --repo-type dataset --commit-message "Initial external data"

pixi run hf upload carpenter-singh-lab/myproject ./data/raw raw/ \
  --repo-type dataset --commit-message "Initial raw data"

pixi run hf upload carpenter-singh-lab/myproject ./data/interim interim/ \
  --repo-type dataset --commit-message "Initial interim data"

pixi run hf upload carpenter-singh-lab/myproject ./data/processed processed/ \
  --repo-type dataset --commit-message "Initial processed results"
```

**Path mapping**: If your local directory layout doesn't match the desired repo
layout, use `path_in_repo` to remap. For example, if your project stores
profiles at `data/raw/profiles/` but the repo should have them at `profiles/`:

```bash
pixi run hf upload carpenter-singh-lab/myproject ./data/raw/profiles profiles/ \
  --repo-type dataset --commit-message "Initial profiles"
```

For very large initial uploads, use `upload_folder` (Python API) or
`upload-large-folder` (CLI, resumable across interruptions):

```bash
pixi run hf upload-large-folder carpenter-singh-lab/myproject ./data/raw \
  --repo-type dataset --num-workers 16
```

Note: `upload-large-folder` always uploads to the repo root (no
`path_in_repo`), so structure your local directory to match the desired repo
layout.

### 6. Branching model

**Same branch name on GitHub and HF Hub.** The HF branch is derived
automatically from your current git branch:

```text
GitHub: main                    ←→  HF Hub: main
GitHub: feat/batch-correction   ←→  HF Hub: feat/batch-correction
```

This means:

- When you `git checkout -b feat/thing`, run `just branch-create` to create
  the matching HF branch. (This could be automated with a `post-checkout` git
  hook — worth adding if people keep forgetting.)
- `just put-results` always pushes to the HF branch matching your current git
  branch. It refuses to push if you're on `main` (use `put-results-main` or
  `pr-create` instead).
- `just get-results-from feat/thing` lets a teammate download your in-progress
  data by branch name.
- Commit messages on HF include the current git SHA for cross-repo traceability.

**HF Hub PRs are not branch-to-branch merges.** Unlike GitHub, HF PRs use
`refs/pr/N` refs. `just pr-create` uploads your current results as a new PR
targeting main — it doesn't merge your HF branch into main. The branch is your
working space; the PR is the merge request.

### 7. Justfile

```just
set dotenv-load := true

# ==================== PROJECT CONFIGURATION ====================
# Hugging Face Hub configuration
HF_ORG := "carpenter-singh-lab"
HF_PROJECT := env_var_or_default("HF_PROJECT", "myproject")
HF_REPO := HF_ORG + "/" + HF_PROJECT

# Branch name mirrors current git branch (override with HF_BRANCH env var)
HF_BRANCH := env_var_or_default("HF_BRANCH", `git branch --show-current`)

# Current git short SHA — included in HF commit messages for traceability
GIT_SHA := `git rev-parse --short HEAD`

# Patterns to exclude from uploads (each needs its own --exclude flag)
HF_EXCLUDE := '--exclude "*.DS_Store" --exclude "__pycache__" --exclude "*.pyc" --exclude ".ipynb_checkpoints" --exclude "*.tmp"'

CORES := env_var_or_default("CORES", "all")
SNAKEMAKE := "pixi run snakemake"
HF := "pixi run hf"

# Directory names (unchanged from current workflow)
DATA_DIR := "data"
EXTERNAL_DIR := DATA_DIR + "/external"
RAW_DIR := DATA_DIR + "/raw"
INTERIM_DIR := DATA_DIR + "/interim"
PROCESSED_DIR := DATA_DIR + "/processed"

default:
    @just --list

# ==================== BRANCH MANAGEMENT ====================

# Create your working branch
branch-create:
    @echo "Creating branch '{{HF_BRANCH}}'..."
    {{HF}} repo branch create {{HF_REPO}} {{HF_BRANCH}} --repo-type dataset

# List all branches
branch-list:
    #!/usr/bin/env bash
    echo "Branches on {{HF_REPO}}:"
    pixi run python3 -c "
    from huggingface_hub import list_repo_refs
    refs = list_repo_refs('{{HF_REPO}}', repo_type='dataset')
    for b in refs.branches:
        print(f'  {b.name}')
    "

# ==================== MAIN WORKFLOW ====================

# Run full pipeline
run:
    @echo "Running pipeline with {{CORES}} cores..."
    @{{SNAKEMAKE}} all --cores {{CORES}} --printshellcmds --scheduler greedy

# Preview what will run
dry:
    @{{SNAKEMAKE}} --cores {{CORES}} -n -p --scheduler greedy

# ==================== DATA SYNC ====================

# Download input data from main (shared, read-only)
get-inputs:
    @echo "Downloading inputs from {{HF_REPO}} (main)..."
    @mkdir -p {{EXTERNAL_DIR}} {{RAW_DIR}}
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --include "external/**" --include "raw/**" \
      --local-dir {{DATA_DIR}}

# Download results from main
get-results:
    @echo "Downloading results from {{HF_REPO}} (main)..."
    @mkdir -p {{INTERIM_DIR}} {{PROCESSED_DIR}}
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --include "interim/**" --include "processed/**" \
      --local-dir {{DATA_DIR}}

# Download results from a specific branch
get-results-from branch:
    @echo "Downloading results from branch '{{branch}}'..."
    @mkdir -p {{INTERIM_DIR}} {{PROCESSED_DIR}}
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --revision {{branch}} \
      --include "interim/**" --include "processed/**" \
      --local-dir {{DATA_DIR}}

# Download results from main (clean — deletes local files removed from remote)
get-results-clean:
    @echo "Clean-downloading results from {{HF_REPO}} (main)..."
    rm -rf {{INTERIM_DIR}} {{PROCESSED_DIR}}
    @mkdir -p {{INTERIM_DIR}} {{PROCESSED_DIR}}
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --include "interim/**" --include "processed/**" \
      --local-dir {{DATA_DIR}}

# Download specific analysis results
get-results-for run_path:
    @echo "Downloading results for: {{run_path}}"
    @mkdir -p {{PROCESSED_DIR}}/{{run_path}}
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --include "processed/{{run_path}}/**" \
      --local-dir {{DATA_DIR}}

# Upload results to YOUR BRANCH (safe, isolated)
put-results:
    #!/usr/bin/env bash
    if [ "{{HF_BRANCH}}" = "main" ]; then
        echo "ERROR: Cannot push results directly to main. Use put-results-main or pr-create."
        exit 1
    fi
    echo "Uploading results to branch '{{HF_BRANCH}}'..."
    {{HF}} upload {{HF_REPO}} {{INTERIM_DIR}} interim/ \
      --repo-type dataset --revision {{HF_BRANCH}} \
      {{HF_EXCLUDE}} \
      --commit-message "Update interim results (git: {{GIT_SHA}})"
    {{HF}} upload {{HF_REPO}} {{PROCESSED_DIR}} processed/ \
      --repo-type dataset --revision {{HF_BRANCH}} \
      {{HF_EXCLUDE}} \
      --commit-message "Update processed results (git: {{GIT_SHA}})"

# Upload specific analysis results to your branch
put-results-for run_path:
    #!/usr/bin/env bash
    if [ "{{HF_BRANCH}}" = "main" ]; then
        echo "ERROR: Cannot push results directly to main. Use put-results-main or pr-create."
        exit 1
    fi
    echo "Uploading {{run_path}} to branch '{{HF_BRANCH}}'..."
    {{HF}} upload {{HF_REPO}} \
      {{PROCESSED_DIR}}/{{run_path}} processed/{{run_path}}/ \
      --repo-type dataset --revision {{HF_BRANCH}} \
      {{HF_EXCLUDE}} \
      --commit-message "Update {{run_path}} (git: {{GIT_SHA}})"

# Upload results directly to main (maintainers only)
put-results-main:
    @echo "Uploading results directly to main..."
    {{HF}} upload {{HF_REPO}} {{INTERIM_DIR}} interim/ \
      --repo-type dataset {{HF_EXCLUDE}} \
      --commit-message "Update interim results (git: {{GIT_SHA}})"
    {{HF}} upload {{HF_REPO}} {{PROCESSED_DIR}} processed/ \
      --repo-type dataset {{HF_EXCLUDE}} \
      --commit-message "Update processed results (git: {{GIT_SHA}})"

# Upload results to main as a pull request for review
pr-create message="Merge results":
    @echo "Creating PR from branch '{{HF_BRANCH}}'..."
    {{HF}} upload {{HF_REPO}} ./{{DATA_DIR}} . \
      --repo-type dataset --create-pr \
      --include "interim/**" --include "processed/**" \
      {{HF_EXCLUDE}} \
      --commit-message "{{message}} (git: {{GIT_SHA}})"
    @echo "PR created. Review and merge at:"
    @echo "  https://huggingface.co/datasets/{{HF_REPO}}/discussions"

# ==================== DATA ADMIN (Maintainers Only) ====================

# Upload new input data to main
put-inputs:
    @echo "Uploading inputs to {{HF_REPO}}..."
    {{HF}} upload {{HF_REPO}} {{EXTERNAL_DIR}} external/ \
      --repo-type dataset {{HF_EXCLUDE}} \
      --commit-message "Update external data (git: {{GIT_SHA}})"
    {{HF}} upload {{HF_REPO}} {{RAW_DIR}} raw/ \
      --repo-type dataset {{HF_EXCLUDE}} \
      --commit-message "Update raw data (git: {{GIT_SHA}})"

# Download from original sources then upload to HF
# NOTE: Assumes project has a downloading module — adjust import path per project
get-from-sources:
    @echo "Downloading from original sources..."
    pixi run python -m {{HF_PROJECT}}.downloading.download_data

# ==================== HISTORY & INSPECTION ====================

# Show commit log for your branch
log:
    #!/usr/bin/env bash
    pixi run python3 -c "
    from huggingface_hub import list_repo_commits
    commits = list_repo_commits('{{HF_REPO}}', repo_type='dataset', revision='{{HF_BRANCH}}')
    for c in commits[:20]:
        print(f'{c.commit_id[:8]}  {c.created_at:%Y-%m-%d %H:%M}  {c.title}')
    "

# Show commit log for main
log-main:
    #!/usr/bin/env bash
    pixi run python3 -c "
    from huggingface_hub import list_repo_commits
    commits = list_repo_commits('{{HF_REPO}}', repo_type='dataset')
    for c in commits[:20]:
        print(f'{c.commit_id[:8]}  {c.created_at:%Y-%m-%d %H:%M}  {c.title}')
    "

# Download a specific historical version
get-results-at revision:
    @echo "Downloading results at revision {{revision}}..."
    {{HF}} download {{HF_REPO}} --repo-type dataset \
      --revision {{revision}} \
      --include "interim/**" --include "processed/**" \
      --local-dir {{DATA_DIR}}

# Tag current state of a branch for reproducibility (e.g., just tag paper-v1)
tag name:
    @echo "Tagging '{{HF_BRANCH}}' as '{{name}}'..."
    {{HF}} repo tag create {{HF_REPO}} {{name}} --repo-type dataset --revision {{HF_BRANCH}}
    @echo "Retrieve with: just get-results-at {{name}}"

# ==================== CODE QUALITY ====================

snakelint:
    #!/usr/bin/env bash
    echo "Running snakemake lint..."
    output=$({{SNAKEMAKE}} --lint 2>&1)
    filtered=$(echo "$output" | grep -v -E "(Specify a conda environment|https://snakemake.readthedocs.io|This way, the used software|workflow can be executed on any machine|Also see:$)")
    echo "$filtered" | grep -v -E "^Lints for rule.*:$" | grep -v "^[[:space:]]*$" || echo "No lint warnings!"

# ==================== UTILITIES ====================

status:
    @{{SNAKEMAKE}} --summary

config:
    @echo "Current Configuration:"
    @echo "  HF_ORG: {{HF_ORG}}"
    @echo "  HF_PROJECT: {{HF_PROJECT}}"
    @echo "  HF_REPO: {{HF_REPO}}"
    @echo "  HF_BRANCH: {{HF_BRANCH}}"
    @echo "  GIT_SHA: {{GIT_SHA}}"
    @echo "  CORES: {{CORES}}"

# List files in the repo
list-data:
    #!/usr/bin/env bash
    pixi run python3 -c "
    from huggingface_hub import list_repo_tree
    for item in list_repo_tree('{{HF_REPO}}', repo_type='dataset', revision='{{HF_BRANCH}}'):
        print(f'  {item.path}')
    "

# Purge local HF download cache
cache-clear:
    {{HF}} cache prune
```

______________________________________________________________________

## Daily Workflow

### First day

```bash
git clone https://github.com/broadinstitute/PROJECT_NAME.git
cd PROJECT_NAME
pixi install
pre-commit install --hook-type pre-commit --hook-type pre-push

# Login to Hugging Face (one-time)
pixi run hf auth login

# Get data (from main)
just get-inputs
just get-results
```

**`hf download` does not delete local files removed from the remote.** If a
teammate deletes stale results on main, your local copy silently keeps the old
files. When in doubt, use the clean variant:

```bash
just get-results-clean
```

### Start your day

```bash
git pull                  # Code updates
just get-results          # Latest merged results from HF main
just dry                  # Check pipeline status
```

### Start a feature

```bash
# Create matching branches on GitHub and HF Hub
git checkout -b feat/batch-correction
just branch-create

# Now put-results will target feat/batch-correction on HF Hub
just config               # Verify HF_BRANCH shows your branch name
```

### Work and push

```bash
# Run your analysis...

# Push results to your HF branch (refuses if on main)
just put-results-for my-analysis

# Commit and push code
git add notebooks/3.01-srs-batch-correction.py
git commit -m "feat: batch correction analysis"
git push -u origin feat/batch-correction
```

### Share with the team

```bash
# Let a teammate preview your data branch directly
# (teammate runs:)
just get-results-from feat/batch-correction

# When ready to merge: create PRs on both repos
# GitHub: open PR as usual (web UI or gh pr create)
# HF Hub: upload results as a data PR
just pr-create "Add batch correction results"
```

### When things go wrong

```bash
# See what happened
just log
just log-main

# Download data from a known-good point in time
just get-results-at abc123def456
```

______________________________________________________________________

## Cost

**Team plan**: $20/user/month. Each seat includes 1 TB of private storage.
A 10-person lab = 10 TB included.

**Enterprise plan**: $50/user/month with advanced security, SSO, resource groups.

**Extra storage**: $25/month per additional TB beyond the included amount.

**Academic grants**: HF offers storage grants for impactful research.
Contact <datasets@huggingface.co> — Cell Painting Gallery is exactly the
kind of thing they support. Worth asking.

**Limits**:

- Max 200 GB per file (50 GB recommended shard size)
- Max 10k files per folder
- Max 100k files per repo (split across multiple repos if needed)

______________________________________________________________________

## Trade-offs

This proposal trades one set of problems for another. Worth being explicit
about what we're taking on.

**What HF Hub solves that S3 cannot**: versioning, branching, conflict
prevention, audit trail, chunk-level dedup (Xet), web-browsable data and PRs.

**What HF Hub introduces**:

| Concern | Detail |
| --- | --- |
| Two version control systems | GitHub branches and HF branches must stay in sync. The Justfile handles this, but it's coordination that doesn't exist with a single-repo solution. |
| `hf download` is additive | Downloads add/update files but never delete. If a teammate removes stale results on main, your local copy keeps the old files. Use `get-results-clean` for exact sync. |
| Cost | $20/user/month vs $0 incremental on existing S3. For a 10-person lab this is $2,400/year. |
| Vendor dependency | Data lives on HF infrastructure. If HF changes pricing, rate-limits, or goes away, migration is non-trivial. S3 data is under your control. |
| 100k file limit per repo | Not an issue for profiles/metadata, but Cell Painting image datasets may require repo splitting or archive formats (WebDataset). |

The daily workflow (`just get-results` / `just put-results`) absorbs most of
this complexity. The risk is in the edge cases — stale local files, forgotten
`branch-create`, cache bloat — not in the happy path.

______________________________________________________________________

## FAQ

**Why not DVC?**

We tried it. The problems were daily-use friction, not conceptual:

- **No browsing**: You cannot see what's in the remote without downloading it.
  "What did the pipeline produce?" requires `dvc pull` and waiting.
- **Five-step ceremony per commit**: `dvc add` → `git add *.dvc` → `git commit`
  → `dvc push` → `git push`. Every time.
- **Hooks slow everything down**: DVC installs git hooks to auto-stage `.dvc`
  files. These run on every commit, even commits that don't touch data.
- **File-level dedup only**: DVC re-uploads the entire file if a single byte
  changes. For a 5 GB parquet file with one new column, that's 5 GB uploaded.
  HF Hub's Xet storage does chunk-level dedup — only changed chunks transfer.
- **Pipeline overlap**: DVC has its own pipeline system (`dvc.yaml`/`dvc.lock`)
  that competes with Snakemake. You'd need to choose one or maintain both.

DVC's `.dvc` pointer files do give you automatic branch-level data versioning
(no `branch-create` needed). That's genuinely simpler. But the daily friction
of hooks, ceremony, and inability to browse outweighed it.

**Why not move everything to HF Hub and drop GitHub entirely?**

HF Hub is a great data host, not a development platform. You'd lose:

- **CI/CD**: No GitHub Actions equivalent. No automated tests or linting on push.
- **Code review**: HF PRs use `refs/pr/N` for data commits — no inline code suggestions, thread resolution, or CODEOWNERS.
- **Issue tracking**: HF Discussions are basic. No project boards, labels, or milestones.
- **Ecosystem**: No pre-commit.ci, Codecov, Dependabot, or GitHub Apps.
- **Branch protection**: Minimal compared to GitHub's required reviews and status checks.

The right split: **GitHub for code, HF Hub for data.**

**Is HF Hub designed for arbitrary project data, not just ML models/datasets?**

Technically, dataset repos can hold anything. HF won't police what you put in a
private repo. That said, you lose HF's Data Studio viewer for non-standard
formats. PNGs and Parquet will preview nicely; your DocDB files obviously won't.

**What about the 100k file limit per repo?**

Cell Painting image datasets can be enormous. If you have millions of PNGs, you'd
need to split across repos or use formats like WebDataset (tar archives of
images). For profiles and metadata this is unlikely to be an issue.

**What about bandwidth / download speed?**

Xet's chunk-based transfers are fast. For initial large downloads, performance
should be comparable to or better than rclone from S3, especially for subsequent
syncs where only changed chunks transfer.

**We already pay for S3 — is this double-paying?**

Yes, you'd be paying HF instead of (or in addition to) AWS for storage. But
you're buying versioning, branching, dedup, and a managed service. Evaluate
whether the operational simplicity and safety are worth the cost delta.

**Can we still access data programmatically?**

Not directly from S3 — data lives on HF's infrastructure. But `hf_hub_download`
is fast and caches locally:

```python
import pandas as pd
from huggingface_hub import hf_hub_download

path = hf_hub_download(
    repo_id="carpenter-singh-lab/myproject",
    filename="interim/profiles.parquet",
    repo_type="dataset",
)
df = pd.read_parquet(path)
```

**What about local cache eating disk space?**

`hf download` caches everything under `~/.cache/huggingface/`. This grows
silently. Run `just cache-clear` periodically, or set `HF_HOME` to a location
with more space:

```bash
export HF_HOME=/scratch/$USER/.cache/huggingface
```

**How do I authenticate on a cluster or in CI?**

Set `HF_TOKEN` as an environment variable instead of interactive login:

```bash
# In your SLURM job script or CI config
export HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx
```

Generate a token at <https://huggingface.co/settings/tokens> (read access is
sufficient for downloading; write access for uploads).

______________________________________________________________________

## Rollout Plan

1. **Week 1**: Upload one project's data to <https://huggingface.co/carpenter-singh-lab>. Test the Justfile with 2 people.
2. **Week 2**: Run a real pipeline cycle — get inputs, run snakemake, push results on branches, merge.
3. **Week 3**: If it works, keep old S3 as read-only backup. New work goes through HF.
4. **Month 2**: Migrate remaining active projects. Decommission S3 sync for those projects.

Keep the S3 bucket as a backup during the transition. Don't burn bridges until
the team is comfortable.

______________________________________________________________________

## Open Questions (at time of withdrawal)

- What data layers actually need versioning? (`processed/` only? `interim/` too?)
- Is there a lighter-weight solution that adds browsability and audit trail to
  S3 without replacing the sync layer?
- Could HF Hub be used selectively — e.g., publish final datasets for external
  sharing — without replacing the internal workflow?
