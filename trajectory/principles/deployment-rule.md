
# Standard “Productionization” Finish: `python -m <package> <pipeline>`

This is the last-mile checklist I apply to **crystallize** a data system into something that can run from terminal + scheduler reliably, and scale to more datasets without turning into spaghetti.

## Goal

Turn a working pipeline (e.g., `run_so()`) into a **stable, production-callable interface**:

- Local dev: `python -m path_adapter so --date 2026-01-27`
- Automated: Windows Task Scheduler / cron calls the same command
- Scales: Adding `po`, `inv`, `gl`, etc. becomes a new subcommand, not a new “random script”

## Step 0 — Preconditions (you already have these)

You have:
- A pipeline function like `run_so()` that does end-to-end work
- A clear pipeline boundary (Bronze → Silver → Gold)
- Deterministic outputs (tables / files) and a meaningful success printout/log

Target end state:
- All production execution begins at **one CLI entrypoint**
- Everything else is imported and called from there

## Step 1 — Decide your “Package Name” and Command Shape

**Convention:**
- Package name: `path_adapter`
- Command shape: `python -m path_adapter <pipeline> [options]`

Example commands:
- `python -m path_adapter so`
- `python -m path_adapter so --date 2026-01-27`
- `python -m path_adapter so --input zheSO-2026-1-27.csv`
- `python -m path_adapter so --dry-run`

## Step 2 — Implement `__main__.py` as the Single Entry Point

Create:

`src/path_adapter/__main__.py`

Responsibilities of `__main__.py`:
- Parse CLI arguments
- Route to the correct pipeline function
- Pass parameters (date, input override, dry-run, env)
- Exit with proper status codes (0 success, non-zero failure)

**Minimum skeleton:**

- Use `argparse`
- Use subcommands (`so`, `po`, `inv`, …)
- Default date = `today` if not specified
- Keep it thin: it should *not* implement business logic

## Step 3 — Standardize Pipeline Function Signatures

Upgrade pipeline functions to accept “production parameters” **even if some are ignored initially**.

Recommended signature pattern:

- `run_<pipeline>(run_date, input_file=None, dry_run=False, env="prod")`

Example:
- `run_so(run_date=..., input_file=..., dry_run=..., env=...)`

Why:
- Prevents CLI logic from leaking into the pipeline
- Makes pipelines consistent across datasets
- Makes scheduling/automation uniform

## Step 4 — Add “Production Options” (Pick What You Need)

### Option A — `--date`
Purpose:
- Re-run historical date
- Debug yesterday without changing code

Rule:
- Default to today if missing

### Option B — `--input`
Purpose:
- Override auto file naming or path detection
- Emergency recovery when file name is non-standard

### Option C — `--dry-run`
Purpose:
- Validate parsing + transformations without writing outputs
- Great for testing in production-like environment without side effects

### Option D — `--env {dev,prod}`
Purpose:
- Use different outputs / db / directories
- Avoids “oops wrote to prod” disasters

## Step 5 — Logging (Minimum Production Bar)

### Minimum (must-have)
- Print start/end banners
- Print row counts at each stage
- Print input file name and output destinations
- Return non-zero exit code on failure

### Better (recommended)
- Use Python `logging`
- Write logs to:
  - console (for manual runs)
  - log file per run (for scheduler runs)

Naming convention:
- `logs/<pipeline>/<pipeline>-YYYY-MM-DD.log`

## Step 6 — Write a Run Manifest (Lightweight Observability)

Create a small JSON written at the end of the run:

`manifests/so-2026-01-27.json`

Include:
- pipeline name
- run timestamp
- run_date
- input filename
- silver row count
- gold row count
- output paths
- status: success/failure
- error message if failed

Why this matters:
- You can debug without opening logs
- You can show “freshness” and “last successful run” to stakeholders

## Step 7 — Make It Runnable from a Clean Environment

### The “clean run” requirement
Your pipeline must run when executed from repo root with only:

- a virtual environment activated
- required env vars present
- the CLI command

No notebooks. No manual path hacks. No “it works only in my IDE”.

Test command:
- `python -m path_adapter so`

## Step 8 — Installable Execution (Optional but Very Clean)

If you’re using `pyproject.toml`, add a console script entrypoint:

- `path-etl so ...`

This is optional, but a strong finish when the project is stable.

## Step 9 — Scheduler Integration (Production Trigger)

### Windows Task Scheduler (typical for your setup)

Preferred pattern:
- A `.bat` wrapper that:
  1. activates venv
  2. sets env vars
  3. runs `python -m path_adapter so`

Why:
- Scheduler environments are “sterile”
- Your script must set up everything explicitly

Minimum scheduler guarantees:
- Stable start directory (repo root)
- Stable Python interpreter (venv python)
- Logs written to file

## Step 10 — Final “Ship Checklist” (Do Not Skip)

- `python -m <package> <pipeline>` runs from repo root
- CLI supports at least: `--date`, `--input`, `--dry-run`
- Non-zero exit code on failure
- Row counts printed (or logged) for each stage
- Outputs are atomic (no partial/corrupted writes)
- One-run log file exists (for scheduler debugging)
- (Recommended) Run manifest JSON written at end
- Scheduler calls the CLI (not internal functions)
- README has “How to run” and “How to schedule”


## Scaling Rule: Adding New Datasets

When adding a new dataset pipeline:
1. Implement `run_xx()` (same signature pattern)
2. Add a subcommand `xx` in `__main__.py`
3. Done — now it’s schedulable and standardized

No new random scripts.


## Design Principle

**One command to run the system.**  
Everything else is internal structure.

If I can’t run it with a single CLI command, it’s not production-ready — it’s still a prototype.

