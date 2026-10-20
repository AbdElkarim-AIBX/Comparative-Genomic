# Comparative-Genomic

A step-by-step, **zero-surprise guide** to running **zDB** — a comparative bacterial
genomics platform — in this repository on **GitHub Codespaces** (conda mode, no Docker
needed).

> The full project lives in [`zDB/`](./zDB). Complete docs:
> [zDB README](./zDB/README.md) · [website tutorial](https://zdb.readthedocs.io/en/latest/)

**What zDB does:** upload GenBank files → get ortholog groups (Diamond/MCL), core-gene
phylogenies (MAFFT + FastTree), COG / KEGG / PFAM / SwissProt annotation, AMR + virulence
genes — all browsable in a Django web app on port **8080**.

---

# Table of contents

1. [Verify your environment](#step-0-verify-your-environment)
2. [Set up the base database](#step-1-set-up-the-base-database-once)
3. [Run the analysis](#step-2-run-the-analysis-pipeline)
4. [Launch the web interface](#step-3-launch-the-web-interface)
5. [Common problems & instant fixes](#step-4-debugging--common-issues)
6. [Recommended usage settings for this system](#recommended-usage-settings)

---

## Step 0: Verify your environment

### Requirements

| Tool | Minimum version | What it's for |
|------|-----------------|---------------|
| Nextflow | ≥ 24.04 | pipeline orchestration |
| Conda / Mamba | any recent | dependency management |
| Xvfb | any | virtual X display for tree rendering |
| git | any | cloning |

> ⚠️ Docker or Singularity are **not required** here. In this Codespace, **use conda**
> for everything; Docker/Singularity will fail.

### Install once (already done ✅ in this workspace)

```bash
# 1. Install missing host package
sudo apt-get update && sudo apt-get install -y xvfb    # only Debian/Ubuntu

# 2. Create the main Nextflow conda environment
cd zDB
conda env create -p ./env -f conda/main.yaml    # nextflow + tooling
conda activate ./env

# verify Nextflow is detected
nextflow -version
```

> ⚠️ If `nextflow` is not found, you **must** activate the env created above:
> `conda activate ./env` or `export PATH=$PWD/env/bin:$PATH`

### ✅ Sanity checks

```bash
nextflow -version    # shows 25.10.4
which Xvfb           # /usr/bin/Xvfb
ls refs/ | head      # 5 GenBank files visible
```

---

## Step 1: Set up the base database (once!)

> 🔧 **This step only needs to be run once.** Without it, step 2 will fail with:
> `cannot stat 'zdb_base'`  ← the most common mistake!

```bash
cd zDB
export PATH=$PWD/env/bin:$PATH

nextflow run db_setup.nf \
    --setup_base_db \
    --base_db=$PWD/zdb_ref \
    --docker=false --singularity=false --conda=true
```

**What it does:**

- builds a minimal "skeleton" SQLite database in `zdb_ref/base/zdb_base`
  (contains COG / KEGG definitions; ~5 MB)
- takes ~4 minutes on a 2-core machine

**Verify it worked:**

```bash
ls zdb_ref/base/zdb_base     # ← must exist or step 2 will fail!
# 5246976 zdb_base
```

---

## Step 2: Run the analysis pipeline

Choose either `input.csv` (repo root) that lists the test Chlamydia genomes in `refs/`
or use any folder with GenBank `.gbk` files.

Direct command with parameters:

```bash
cd zDB
export PATH=$PWD/env/bin:$PATH

nextflow run annotation_pipeline.nf \
    --input=input.csv \
    --name=chlam_run \
    --base_db=$PWD/zdb_ref \
    --n_cpus=2 --mem=6G \
    --docker=false --singularity=false --conda=true
```

### Preset options (mutable pipeline parameters)

```bash
# this workspace (2 CPUs): scale --cpu/--mem to your machine if different

--cpu=2                        # parallel process limit (default 16 → use 2 on this Codespace)
--mem=6G                       # max memory (default 8GB → use 6G here)
--amr / --cog / --ko / --pfam / --gi / --vfdb / --swissprot / --refseq   # add extra analyses
--num_missing=0                # genomes allowed to miss for core orthogroup detection
--og_seq_aligner=diamond       # blast or diamond (default diamond)
--og_n_core=64                 # minimum number of copies to call an orthogroup a core gene
```

**⏱️ Expected duration**: ~30–60 min for 5 genomes on 2 CPUs with only the default
analyses (no --pfam, --swissprot, etc.). Test first, add analyses later.

### Monitoring progress

```bash
# (run these from inside the zDB/ folder)
cd zDB

# number of pipeline tasks completed so far:
ls work | wc -l

# which process stages have run:
ls -t zdb/results/pipeline_info/execution_trace_*.txt | head -1 |
    xargs awk 'NR>1{print $4}' | sort | uniq -c | sort -rn | head -20
```

### What goes into `zdb/results/`

```
zdb/results/
├── db/<run_name>/          # the run's SQLite database (what the webapp reads)
├── blast_DB/<run_name>/    # BLAST indexes for homology queries in the webapp
├── pipeline_info/          # execution report / timeline / DAG / trace (HTML debugging aids)
└── .completed_runs/        # registry of finished runs (also defines the 'latest' alias)
```

When the workflow shows **✔ 100% succeeded** and "Annotation pipeline completed",
**you are done** — go to step 3.

### If the run is interrupted

Forget everything else. Just re-run the **same** command with `-resume` added.
Completed stages will be cached — only the incomplete parts continue.

```bash
nextflow run annotation_pipeline.nf --input=input.csv \
    --name=chlam_run \
    --base_db=$PWD/zdb_ref --n_cpus=2 --mem=6G \
    --docker=false --singularity=false --conda=true \
    -resume
```

---

## Step 3: Launch the web interface

```bash
cd zDB
export PATH=$PWD/env/bin:$PATH

# Optional (Codespaces only): make the forwarded port public so the link works anywhere
gh codespace ports visibility 8080:public -c $CODESPACE_NAME

# The first launch builds the 'webapp' conda environment — takes a few minutes(!),
# it is cached afterwards. --allowed_host/--trusted_origins are REQUIRED on
# Codespaces: Django would otherwise reject the *.app.github.dev hostname.
bash bin/zdb webapp \
    --name=chlam_run \
    --conda --dir=$(pwd) \
    --allowed_host=.app.github.dev \
    --trusted_origins="https://${CODESPACE_NAME}-8080.app.github.dev"
```

> ⚠️ `--name` here **must match** the `--name` you used in step 2!
> If you never passed a `--name` in step 2, the webapp defaults to `--name=latest`,
> which points to the last run marked complete. To see all available runs:

```bash
bash bin/zdb list_runs
```

**How to connect:**

- On **GitHub Codespaces**: click the `Ports` panel, select port `8080` and
  "Open in Browser". The auto-forwarded URL looks like:

  ```text
  https://expert-chainsaw-pj4pqjqg5v4427vx7-8080.app.github.dev
  ```

  (replace the subdomain with **your** own Codespace name)

  Optional: make the forwarded port publicly reachable so the link opens from anywhere
  (see `gh codespace ports visibility --help`):

  ```bash
  gh codespace ports visibility 8080:public -c $CODESPACE_NAME
  ```

- On **any container** that has the pipeline data mounted and exposes port 8080.

To stop it: `Ctrl+C` in that terminal.

---

## Step 4: Debugging & common issues

*(All fixes below are relevant for conda mode in this repo; if you use
Docker/Singularity elsewhere, your mileage may vary.)*

| Symptom | Reason | Fix |
|---------|--------|-----|
| "cannot stat 'zdb_base'" | Step 1 was skipped | Run step 1, then resume step 2 with `-resume` |
| "nextflow: command not found" | conda env not activated | `export PATH=$PWD/env/bin:$PATH` |
| `Error: FileAlreadyExistsException: execution_report_...` | report/timeline filenames collided (same-second rerun) | just re-run |
| `Could not instantiate Xvfb` / display errors | Xvfb is not installed | `sudo apt-get install -y xvfb` |
| conda env build fails or hangs | flaky channel/network | re-run the same command with `--resume` (envs are cached per-task) |
| "run out of space" | `work/` accumulates outputs of all runs | clean up: `nextflow clean -before <run_name> -f` (removes older runs), or `rm -rf work` when you won't resume anything |
| webapp shows blank page | results missing because step 2 was actually incomplete | rerun step 2 with `-resume` and verify 100% completion |
| "A run with that name already exists" | you are re-declaring an already-created run | use a new name: `--name=my_run2` |
| "Permission denied" on a file/dir | repo file access | `sudo chown -R $(whoami) zDB/` |

---

## Recommended usage settings

```bash
# Optimum for this 2-CPU / 8GB Codespace
nextflow run annotation_pipeline.nf --input=input.csv --name=chlam_run \
    --base_db=$PWD/zdb_ref --n_cpus=2 --mem=6G \
    --docker=false --singularity=false --conda=true
```

```bash
# Optimum for the webapp (Codespaces: pass the allowed-host/CSRF flags as in step 3)
bash bin/zdb webapp --name=chlam_run --conda --dir=$(pwd) \
    --allowed_host=.app.github.dev \
    --trusted_origins="https://${CODESPACE_NAME}-8080.app.github.dev"
```

```bash
# Full environmental paths / variables (put in ~/.bashrc to always rely on default state)
export PATH=/workspaces/Comparative-Genomic/zDB/env/bin:$PATH
export ZDB_HOME=/workspaces/Comparative-Genomic/zDB
```

**When to clean up**

- After a run you no longer need: `nextflow clean -after <run_name> -f` (dry-run first with `-n`!)
- `nextflow clean -f` after selecting a run also frees `work/`, but kills the resume cache
- `rm -rf work` — **not safe to do casually**: it deletes the resume cache, forcing everything to recompute

## Maintaining the test dataset

zDB's test dataset ships with the repository; verify it is present:

```bash
ls refs/*.gbk | wc -l       # 5 files = ready for a fresh test run
```

If files are missing:

```bash
tar -xzf test_dataset.tar.gz   # restores refs/*.gbk and the input.csv file
```

## Further documentation

- [zDB README](./zDB/README.md) — full usage reference (setup, run, webapp, export, FAQ)
- [FAQ](./zDB/FAQ.txt) — frequently asked questions
- [CHANGELOG](./zDB/CHANGELOG.md) — release history
- [Testing](./zDB/testing/) — test suite and dataset details

---

> **TL;DR for the impatient** (from repo root):
>
> ```bash
> cd zDB && export PATH=$PWD/bin:$PWD/env/bin:$PATH
> nextflow run db_setup.nf --setup_base_db --base_db=$PWD/zdb_ref \
>     --docker=false --singularity=false --conda=true       # once
> nextflow run annotation_pipeline.nf --input=input.csv --name=chlam_run \
>     --base_db=$PWD/zdb_ref --n_cpus=2 --mem=6G \
>     --docker=false --singularity=false --conda=true       # the analysis
> zdb webapp --name=chlam_run --conda --dir=$(pwd) \
>     --allowed_host=.app.github.dev \
>     --trusted_origins="https://${CODESPACE_NAME}-8080.app.github.dev"  # the webapp
> ```
>
> Then open the forwarded port **8080** in the Codespace Ports panel. 🚀
