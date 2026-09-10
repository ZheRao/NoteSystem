# the Company: From Manual Pipeline to Automated Deployment
## An architecture and operations reference

**Scope.** How the three parts of the Company (input configuration, forecast engine, serving layer) fit together as an automated system; what database technology belongs in each role; what to deploy on Azure and what it costs; and the operational questions that don't announce themselves until they bite.

**Audience.** You, six months from now, when you've forgotten why the artifact directory has a `current` symlink in it.

**How to read this.** Part 1 and Part 2 are the load-bearing parts — the reframe and the invariants. Everything after is elaboration. If you read nothing else, read those two and Part 9 (the concrete recommendation).

---

# Table of Contents

**Foundations**
- Part 0 — The short answer
- Part 1 — The reframe: three roles, not three databases
- Part 2 — The invariants

**Storage**
- Part 3 — Choosing a database per role
- Part 4 — The serving database as a build artifact
- Part 5 — Atomic publication and rollback

**Automation**
- Part 6 — Four levels of orchestration
- Part 7 — The job table pattern
- Part 8 — Failure modes

**Deployment**
- Part 9 — Azure topologies and cost
- Part 10 — The recommended stack, concretely

**Operations**
- Part 11 — Backups, secrets, logging, alerting
- Part 12 — Security and multi-tenancy
- Part 13 — Migrations and contract versioning
- Part 14 — Environments and testing

**Closing**
- Part 15 — Questions you didn't know to ask
- Part 16 — Sequencing
- Appendix A — Cost reference
- Appendix B — Decision triggers
- Appendix C — Glossary

---

# Part 0 — The short answer

Before the reasoning, the conclusions, so you can read the rest knowing where it lands.

**On database choice.** You don't have one database problem, you have two, and they have opposite requirements. The input configuration store wants transactions, constraints, and durability. The serving store wants to be read-only, rebuilt wholesale, and swapped atomically. Postgres is a good answer for the first. For the second, Postgres is *acceptable* but SQLite is arguably *better*, because a serving database that is rebuilt from scratch on every run is not really a database — it's a build artifact, and SQLite artifacts are single files you can version, checksum, copy, and roll back with a symlink.

**On the SQLite-equivalent question.** The most suitable database for your initial deployment may well be the one you already have. SQLite's weaknesses (one writer at a time, no network access, weak typing) barely touch a system where one analyst edits configuration and everyone else reads derived numbers. Its strengths (zero operational surface, file-as-artifact, trivially fast reads) map almost perfectly onto your architecture. The honest recommendation is a hybrid, and Part 3 lays out when to move.

**On automation.** Don't buy an orchestrator. Put a `forecast_runs` table in your configuration database, have publishing a revision insert a row into it, and have a long-lived worker process poll that table and run the engine. This is the outbox pattern. It costs you about 150 lines, adds zero infrastructure, gives you an audit trail and retries for free, and is inspectable with `SELECT`. It is also the same shape as a real orchestrator, so if you outgrow it the concepts transfer.

**On cloud services.** One Linux VM in Canada Central running everything, behind Caddy for TLS, with the artifact directory backed up to Azure Blob Storage. Roughly $25–45/month. Add managed Postgres later if and when the configuration store's durability or concurrency needs exceed what you want to operate yourself. Don't start with Container Apps, Kubernetes, or a managed database per component — every one of those adds a failure mode you'll have to learn to debug alone.

**On the unknown unknowns.** The three that matter most, in order: (1) you have no tested restore procedure, (2) you have no story for client #2 and the decision you make now determines how painful that is, and (3) a forecast that is silently *wrong* is far more dangerous than one that crashes, and nothing in your current pipeline distinguishes them. Parts 11, 12, and 14 respectively.

---

# Part 1 — The reframe: three roles, not three databases

The most useful thing you can do before picking technology is stop thinking about "the database" and start thinking about **three distinct data roles** with different lifecycles, different consistency requirements, and different disaster stories.

## 1.1 The three roles

**Role A — System of record (the configuration store).**
This is the only data in your system that cannot be recreated. If the forecast engine's output vanishes, you rerun it. If a client's input configuration vanishes, you call the client and apologize. Everything about how you treat this store follows from that asymmetry.

Characteristics: small (megabytes), mutable, transactional, constraint-heavy, low write volume, high value per byte. You've already built it correctly — FK constraints, CHECK constraints, draft/published revision lifecycle enforced by triggers, an append-only audit log.

**Role B — Compute input (the engine's read of a revision).**
This is not a store at all. It's a *snapshot* — a materialized read of one published revision, loaded into memory, fed to the forecast engine. It has no lifecycle of its own. Its only requirement is that it be a **consistent point-in-time read**: the engine must never see half of revision 7 and half of revision 8.

Characteristics: ephemeral, read-once, must be internally consistent, must be *identified* (you need to know exactly which revision produced which output).

**Role C — Serving artifact (the read-only forecast database).**
This is derived data. It is a *function* of (configuration revision, engine version). It is written once by a batch process and then never modified. Many readers, zero writers. If it is lost or corrupted, you rebuild it from Role A.

Characteristics: immutable after build, disposable, rebuildable, read-heavy, analytically shaped (your drill-down aggregations), versioned.

## 1.2 Why this framing pays off

Because the moment you name Role C as *derived and disposable*, a pile of hard problems evaporate:

- **You don't need to migrate it.** Schema changes to the serving database are not migrations — they're a new build. Drop the old artifact, build a new one. Compare that to migrating a live transactional database with client data in it.
- **You don't need to back it up.** Back up Role A and your code. Role C is reproducible.
- **You don't need transactions in it.** Nobody writes to it concurrently. There is exactly one writer, and it runs alone.
- **You don't need referential integrity enforcement at runtime.** You validate at *build* time — which you already do, with your data contract and manifest verification — and then the artifact is frozen.
- **Rollback becomes trivial.** Rolling back a live database means restoring a backup and losing writes. Rolling back an artifact means pointing at the previous file.

You already know this pattern. In ETL terms, Role A is your source system, Role B is your extract, Role C is your published mart. The mart is rebuilt, not migrated. What's new here is that the mart is *also* the thing serving live HTTP traffic, which is why publication needs to be atomic (Part 5).

## 1.3 The shape of the whole system

```
                   ┌──────────────────────────────────────────┐
                   │  ROLE A — Configuration store            │
   Analyst  ──────▶│  mutable · transactional · backed up     │
   (write)         │  revisions: draft → published            │
                   │  audit log (append-only)                 │
                   │  forecast_runs (job queue)               │
                   └────────────┬─────────────────────────────┘
                                │  publish event enqueues a run
                                ▼
                   ┌──────────────────────────────────────────┐
                   │  WORKER (long-lived process)             │
                   │  claims run → ROLE B snapshot read       │
                   │  → forecast engine → build artifact      │
                   │  → verify against data contract          │
                   │  → atomic publish                        │
                   └────────────┬─────────────────────────────┘
                                │  writes + swaps
                                ▼
                   ┌──────────────────────────────────────────┐
                   │  ROLE C — Serving artifacts              │
                   │  /artifacts/2026-09-10T14-22-run117.db   │
                   │  /artifacts/current  ──▶ (symlink)       │
                   └────────────┬─────────────────────────────┘
                                │  read-only
                                ▼
                   ┌──────────────────────────────────────────┐
                   │  FastAPI serving layer  ──▶  React SPA   │
                   └──────────────────────────────────────────┘
```

Note what is *not* in this diagram: the frontend never touches Role A, and the serving API never writes anything. Those are invariants, not accidents. See I8 below.

---

# Part 2 — The invariants

These are the rules that must hold no matter which database, cloud, or orchestrator you pick. If a design choice violates one of these, the choice is wrong, not the invariant. Write them somewhere you'll see them.

**I1 — Exactly one system of record per fact.**
Every fact in the system is authoritative in exactly one place. Configuration facts live in Role A. Derived facts live in Role C and are authoritative nowhere — they are a cache of a computation. If you ever find yourself editing a number in the serving database, you have violated this and the next rebuild will silently revert your edit.

**I2 — Derived state is fully reproducible.**
Given a configuration revision ID and an engine version, you can reproduce a byte-identical (or at minimum, numerically identical) serving artifact. This requires: no wall-clock dependence in the engine except an explicit `as_of` parameter, no reliance on unpinned dependency versions, no unordered iteration affecting float accumulation, no randomness without a seed.

If you can't reproduce yesterday's numbers today, you cannot answer a client asking "why did this change?" — which, for a financial forecasting product, is the question you will be asked most often.

**I3 — Every artifact carries its lineage.**
Every serving artifact has a manifest recording: configuration revision ID, config content hash, engine version (git SHA), build timestamp, engine parameters, row counts, and the results of contract verification. The API exposes this. The frontend displays it. When a client says "these numbers look wrong," the first question is "which build are you looking at," and it must be answerable in five seconds.

**I4 — Publication is atomic.**
A reader sees either the complete old artifact or the complete new one. Never a partially-written file, never a mix. There is no intermediate state visible to anyone.

**I5 — Rollback is a pointer move.**
Reverting to the previous good forecast takes seconds and requires no rebuild, no restore, and no database surgery. If rollback requires thinking, you won't do it under pressure.

**I6 — Jobs are idempotent and keyed.**
Running the same job twice produces the same result and causes no harm. A job is identified by `(revision_id, engine_version)`. Enqueuing a duplicate is a no-op, not a second run.

**I7 — Nothing critical depends on a human remembering.**
Every step that currently begins with "then I run..." is a step that will eventually be skipped, run out of order, or run against stale inputs. Automation here is not about saving your time — it's about eliminating a class of correctness bug.

**I8 — The serving path is read-only and independent.**
The API that serves the frontend has no write access to anything and does not depend on the configuration store being up. If the config database is down for maintenance, clients can still view forecasts. This is the single cheapest availability win in the whole design, and it's free — you get it by not coupling them.

**I9 — Validation is a gate, not a report.**
Your contract checks and reconciliation checks either block publication or they are decoration. A build that fails verification must not become `current`. Ever. Not even "just this once because the client meeting is at 3."

**I10 — Money is exact.**
Never float. `NUMERIC`/`DECIMAL` in the database, `decimal.Decimal` in Python, integer cents if you prefer. Floats accumulate error across your monthly-to-annual aggregations, and your reconciliation checks will start failing by pennies in ways that take days to trace. This is worth fixing before you have client data, not after.

---

# Part 3 — Choosing a database per role

Now the actual question: PostgreSQL, SQLite, or something else, and where.

## 3.1 What Postgres actually buys you

It's worth being precise, because "use Postgres, it's the real database" is cargo-culted advice and half the reasons don't apply to you.

| Postgres capability | Does it matter for the Company *now*? |
|---|---|
| Concurrent writers with row-level locking | **No.** One analyst edits configuration. SQLite's single-writer model is not a constraint you'd notice. |
| Network access from multiple hosts | **Not yet.** Matters the day your API and worker run on different machines. |
| Strong typing, real `NUMERIC`, native dates/arrays/JSONB | **Yes.** You already hit SQLite's affinity and collation edges when exploring join ordering. Weak typing in a financial system is a real hazard. |
| Row-level security | **Later, and importantly.** This is your multi-tenancy lever (Part 12). |
| Managed backups, PITR, HA | **Yes** — this is arguably the biggest single win, because it's operational work you otherwise do yourself. |
| Mature migration tooling (Alembic) | Works fine with SQLite too, but SQLite's limited `ALTER TABLE` makes migrations more painful. |
| Query planner sophistication, real `EXPLAIN ANALYZE` | **Marginal at your data volume.** Farm-scale financial data is small. |
| Extensions (PostGIS, TimescaleDB, `pg_cron`) | **Speculative.** Geospatial for field boundaries is plausible someday. |

The honest summary: for Role A, Postgres wins on **typing, managed durability, and future multi-tenancy** — not on concurrency or performance. Those three are good enough reasons on their own, but they're different reasons than most people give.

## 3.2 What SQLite buys you

| SQLite property | Value for the Company |
|---|---|
| Zero operational surface — no server, no port, no user management, no version upgrades | **Very high** for a solo developer. Every service you run is a service you get paged about. |
| The database *is a file* | **Extremely high for Role C.** Artifact, checksum, copy, symlink, roll back, ship to a client. All of that is `cp` and `ln`. |
| No network round-trip per query | Read latency in microseconds. Your drill-down endpoints get fast for free. |
| Backup is a file copy | Trivial for Role C. Slightly subtler for Role A (use the backup API or `VACUUM INTO`, not `cp`, on a live file). |
| No connection pooling needed | The entire Part-2-of-this-conversation problem doesn't exist. |

And the costs: one writer at a time (fine), weak type affinity (a real problem for Role A, irrelevant for a write-once Role C), no network access (constrains topology), no RLS, and limited `ALTER TABLE`.

## 3.3 DuckDB for Role C — worth a serious look

Your serving workload is: read-only, columnar-friendly aggregations, drill-down from annual to category to source to monthly schedule. That is precisely DuckDB's target. Relative to SQLite for this role:

- **Analytical query performance** on group-bys and window functions is typically an order of magnitude better, because it's columnar and vectorized.
- **Real types**, including `DECIMAL` with proper precision, native `DATE`/`INTERVAL`, `LIST`, `STRUCT`.
- **Reads Parquet natively**, which matters if your engine's natural output is dataframes — you can skip a serialization step.
- Same file-as-artifact property as SQLite.

Costs and cautions:
- **Storage format stability.** DuckDB's on-disk format has historically changed between versions. For an artifact you rebuild every run, this is much less scary than for a system of record — but pin the version and rebuild artifacts on upgrade.
- **Concurrent read access from multiple processes** requires opening in read-only mode; it works, but the model is less battle-tested than SQLite's for many-reader web serving.
- Another dependency and another SQL dialect to hold in your head.

**Verdict:** if your serving queries are currently fast enough in SQLite, stay with SQLite — you have plenty of harder problems. Revisit DuckDB when a drill-down endpoint gets slow or when you add a client whose data is 50× larger. The decision is cheap to revisit precisely *because* the artifact is rebuilt every time; there is no migration.

## 3.4 The three viable configurations

**Configuration 1 — All SQLite (where you are now, operated properly).**

Role A: SQLite with WAL mode, Litestream continuously replicating to Azure Blob Storage. Role C: SQLite artifacts with symlink swap.

- Cost: ~$25/mo (one VM + blob storage).
- Operational load: lowest.
- Weakness: no managed PITR you didn't build, weak typing in the system of record, and a ceiling you'll hit when you want the API and worker on separate machines.
- **Good choice if** your priority is shipping to your first client and the configuration store has a single editor.

**Configuration 2 — Postgres for Role A, SQLite/DuckDB artifacts for Role C. (Recommended.)**

Role A: Azure Database for PostgreSQL Flexible Server, Burstable tier. Role C: file artifacts on the VM's disk, replicated to Blob.

- Cost: ~$40–55/mo.
- You get managed backups and PITR for the data you cannot recreate, real `NUMERIC` and `DATE` types, RLS available for multi-tenancy later — while keeping the artifact model's simplicity for serving.
- The asymmetry is the point: pay for durability where data is irreplaceable, pay nothing where it's rebuildable.
- **Weakness:** two database technologies to know. Given you're already fluent in both, this is a small cost.

**Configuration 3 — All Postgres, with schema swap for Role C.**

Role A and Role C both in Postgres, with the serving data built into a staging schema and swapped in atomically.

- Cost: ~$40–60/mo, one database service.
- The swap works because **Postgres DDL is transactional** — a genuine superpower most databases lack:

```sql
BEGIN;
SET lock_timeout = '5s';
DROP SCHEMA IF EXISTS serving_previous CASCADE;
ALTER SCHEMA serving          RENAME TO serving_previous;
ALTER SCHEMA serving_incoming RENAME TO serving;
COMMIT;
```

Either all three statements take effect or none do. Readers with `search_path = serving` see old or new, never a mix.

- **Weakness:** you lose the file-artifact property. Rollback is another schema rename rather than a symlink flip (still fine). Keeping N previous versions means N schemas and N× the storage in your paid database. Build writes go through the network and through WAL, so they're materially slower than writing a local SQLite file — for a full rebuild of many rows, this is the difference between seconds and minutes.
- **Good choice if** you strongly prefer one technology, or if you're heading toward multiple API instances soon (files on one VM's disk don't scale horizontally; a database does).

## 3.5 Recommendation

Start at **Configuration 2**, with Configuration 1 as a legitimate fallback if you want to defer the Postgres cost by a month or two. The migration path from 1 to 2 is short (your config schema is small and you wrote the importer already), so starting at 1 is not a trap.

Do **not** start at Configuration 3 unless you already know you need horizontal API scaling. It's the most conventional-looking choice and the least suited to a batch-rebuild workload.

---

# Part 4 — The serving database as a build artifact

This is the conceptual centre of the design. Internalize this and most of the automation questions answer themselves.

## 4.1 The mental model

A serving artifact is a **build output**, in exactly the sense that a compiled binary or a Docker image is a build output. It has:

- **Inputs**: a configuration revision, an engine version, engine parameters.
- **A deterministic build process**: your forecast engine.
- **A verification step**: your data contract checks and reconciliation.
- **An immutable, addressable result**: a file with a name that identifies it.
- **A deployment step**: making it `current`.
- **A rollback**: making a previous one `current` again.

You don't migrate a binary. You don't hot-patch a Docker image. You build a new one and deploy it. Treat the serving database identically.

## 4.2 Naming and layout

```
/var/lib/the Company/artifacts/
    ├── run-000114-r7-a3f9c21.db          ← artifact
    ├── run-000114-r7-a3f9c21.manifest.json
    ├── run-000116-r8-a3f9c21.db
    ├── run-000116-r8-a3f9c21.manifest.json
    ├── run-000117-r8-b1e4d02.db
    ├── run-000117-r8-b1e4d02.manifest.json
    └── current -> run-000117-r8-b1e4d02.db
```

The name encodes run ID, configuration revision, and engine git SHA. You should be able to look at a filename and know what produced it without opening anything. Two artifacts with the same `r{n}` and same SHA should be numerically identical (invariant I2) — and that's a property you can actually *test*, which is worth doing occasionally as a determinism check.

## 4.3 The manifest

You already built a manifest-based verification mechanism for your data contract. This extends it into the deployment path. Suggested contents:

```json
{
  "run_id": 117,
  "artifact_filename": "run-000117-r8-b1e4d02.db",
  "artifact_sha256": "9f2c...",
  "contract_version": 3,

  "inputs": {
    "config_revision_id": 8,
    "config_content_hash": "sha256:41ab...",
    "client_id": "pederson-heritage",
    "as_of_date": "2026-09-01"
  },

  "engine": {
    "git_sha": "b1e4d02",
    "version": "0.9.3",
    "python": "3.12.4",
    "key_dependencies": {"pandas": "2.2.3", "numpy": "2.1.1"}
  },

  "build": {
    "started_at": "2026-09-10T14:20:11Z",
    "finished_at": "2026-09-10T14:22:47Z",
    "duration_seconds": 156
  },

  "verification": {
    "status": "passed",
    "checks": [
      {"name": "grain_separation",        "status": "passed"},
      {"name": "fk_integrity",            "status": "passed"},
      {"name": "arithmetic_conservation", "status": "passed", "max_abs_delta": "0.00"},
      {"name": "drill_path_reconciliation","status": "passed", "mismatches": 0},
      {"name": "temporal_validity",       "status": "passed"}
    ]
  },

  "shape": {
    "row_counts": {"cash_flow_streams": 4820, "output": 1104, "input_output_linkage": 9640},
    "months_covered": ["2026-01", "2031-12"]
  }
}
```

Why each part earns its place:

- `artifact_sha256` lets you verify the file wasn't corrupted in transit or on disk, and lets you detect the "same inputs, different output" determinism failure.
- `config_content_hash` catches the nightmare case where someone edits a published revision in place. Revision IDs can lie; content hashes can't.
- `contract_version` is how the API refuses to serve an artifact whose shape it doesn't understand (Part 13).
- `row_counts` and `months_covered` are your cheapest anomaly detector. A build that produces 40% fewer rows than the last one is almost certainly broken even if every check passed.

## 4.4 Serving from an artifact

```python
# app/artifacts.py
import json, os, sqlite3, threading
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class Artifact:
    path: Path
    manifest: dict

class ArtifactHolder:
    """Holds the currently-published artifact. Reloadable without restart."""

    def __init__(self, pointer: Path):
        self._pointer = pointer
        self._lock = threading.Lock()
        self._current: Artifact | None = None
        self.reload()

    def reload(self) -> Artifact:
        target = self._pointer.resolve(strict=True)
        manifest_path = target.with_suffix(".manifest.json")
        manifest = json.loads(manifest_path.read_text())

        if manifest.get("verification", {}).get("status") != "passed":
            raise RuntimeError(f"refusing to serve unverified artifact {target.name}")
        if manifest.get("contract_version") not in SUPPORTED_CONTRACT_VERSIONS:
            raise RuntimeError(f"unsupported contract version in {target.name}")

        artifact = Artifact(path=target, manifest=manifest)
        with self._lock:
            self._current = artifact
        return artifact

    @property
    def current(self) -> Artifact:
        with self._lock:
            if self._current is None:
                raise RuntimeError("no artifact published")
            return self._current

    def connect(self) -> tuple[sqlite3.Connection, Artifact]:
        art = self.current
        # mode=ro: read-only. immutable=1: file will never change while open,
        # which is guaranteed because artifacts are write-once. Lets SQLite
        # skip all locking — meaningfully faster.
        conn = sqlite3.connect(
            f"file:{art.path}?mode=ro&immutable=1",
            uri=True,
            check_same_thread=False,
        )
        conn.row_factory = sqlite3.Row
        return conn, art
```

Wired into FastAPI:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.artifacts = ArtifactHolder(Path(settings.artifact_pointer))
    yield

def get_db(request: Request):
    conn, artifact = request.app.state.artifacts.connect()
    try:
        yield conn, artifact
    finally:
        conn.close()

@app.get("/api/forecast/annual")
def annual(db = Depends(get_db)):
    conn, artifact = db
    rows = conn.execute("SELECT ...").fetchall()
    return {
        "data": [dict(r) for r in rows],
        "build": {                       # invariant I3, surfaced to the UI
            "run_id":      artifact.manifest["run_id"],
            "revision_id": artifact.manifest["inputs"]["config_revision_id"],
            "built_at":    artifact.manifest["build"]["finished_at"],
        },
    }
```

Put the build stamp in *every* response payload, not just a `/health` endpoint, and render it somewhere in the UI. When a client screenshots a number and asks about it, the screenshot then contains its own provenance.

## 4.5 Retention

Keep the last N artifacts (5–10 is plenty). Delete older ones on a schedule, never immediately after a swap — an in-flight request may still hold an open file descriptor. On Linux, an open fd keeps the inode alive even after unlink, so deletion is *safe*, but keeping a few around gives you instant rollback, which is the real reason.

Prune with a rule like "keep the 10 newest, plus whatever `current` points at, plus anything referenced by a run row in the last 30 days."

---

# Part 5 — Atomic publication and rollback

Invariant I4 says readers see old or new, never a mix. Here's how you actually get that.

## 5.1 Why the naive approach breaks

Writing directly to `serving.db` while the API reads it produces, in order of increasing subtlety: readers seeing partially-written data; readers holding locks that block the writer; a crash mid-write leaving a corrupt file with no good copy; and — worst — a *successful* write of data that fails verification, now live with no way back.

## 5.2 The mechanism: build aside, verify, then flip a pointer

```python
# worker/publish.py
import os, tempfile
from pathlib import Path

ARTIFACT_DIR = Path("/var/lib/the Company/artifacts")
POINTER      = ARTIFACT_DIR / "current"

def publish(run_id: int, built_db: Path, manifest: dict) -> Path:
    """Move a verified build into place atomically."""
    if manifest["verification"]["status"] != "passed":
        raise RuntimeError("refusing to publish an unverified artifact")   # I9

    final = ARTIFACT_DIR / manifest["artifact_filename"]

    # 1. Move the built file into the artifact directory.
    #    os.replace is atomic when source and destination are on the same
    #    filesystem — build in a temp dir on the SAME volume, not /tmp.
    os.replace(built_db, final)

    # 2. Write the manifest next to it, also atomically.
    manifest_final = final.with_suffix(".manifest.json")
    with tempfile.NamedTemporaryFile("w", dir=ARTIFACT_DIR, delete=False) as f:
        json.dump(manifest, f, indent=2)
        f.flush()
        os.fsync(f.fileno())
        tmp_manifest = Path(f.name)
    os.replace(tmp_manifest, manifest_final)

    # 3. Flip the pointer atomically. You cannot atomically "retarget" a
    #    symlink, but you CAN atomically rename one over another.
    tmp_link = ARTIFACT_DIR / f".current.{os.getpid()}.tmp"
    tmp_link.symlink_to(final.name)          # relative target: survives moves
    os.replace(tmp_link, POINTER)            # atomic swap

    # 4. Durability: fsync the directory so the rename survives a crash.
    dir_fd = os.open(ARTIFACT_DIR, os.O_RDONLY)
    try:
        os.fsync(dir_fd)
    finally:
        os.close(dir_fd)

    return final
```

The load-bearing fact: `rename(2)` on POSIX is atomic within a filesystem. At no instant does the pointer not exist or point somewhere invalid. A reader either resolves it to the old file or the new one.

**Gotcha:** if `built_db` is in `/tmp` and your artifact directory is on a different mount, `os.replace` falls back to a non-atomic copy. Build into a `staging/` subdirectory of the artifact directory itself.

## 5.3 Telling the API to reload

The pointer moved; the running API still has the old artifact cached. Three options, cheapest first:

1. **Resolve the symlink per request.** Costs one `readlink` syscall — genuinely negligible. Simplest correct thing.
2. **Signal-based reload.** Worker sends `SIGHUP`; the API's handler calls `holder.reload()`. Clean, no polling, no HTTP surface.
3. **Internal HTTP endpoint.** `POST /internal/reload`, bound to localhost only, called by the worker. Easiest to debug, but it's a mutating endpoint on a service you declared read-only (I8) — bind it to loopback and don't expose it.

I'd start with (1) and move to (2) if the syscall ever shows up in a profile, which it won't.

In-flight requests holding an open fd to the old artifact continue against it and finish correctly. That's the desired behaviour: a single request never sees two different builds mid-flight.

## 5.4 Rollback

```bash
#!/usr/bin/env bash
# rollback.sh — point at a previous artifact. Should take under five seconds.
set -euo pipefail
ART=/var/lib/the Company/artifacts
TARGET="${1:?usage: rollback.sh <artifact-filename>}"

test -f "$ART/$TARGET" || { echo "no such artifact"; exit 1; }
ln -sfn "$TARGET" "$ART/.current.tmp"
mv -Tf "$ART/.current.tmp" "$ART/current"
systemctl reload the Company-api
echo "now serving: $(readlink "$ART/current")"
```

Write this script *now*, before you need it. Run it once in staging so you know it works. A rollback procedure discovered during an incident is not a rollback procedure.

## 5.5 The Postgres equivalent

If you go with Configuration 3, the same guarantees come from transactional DDL:

```python
def publish_pg(conn, incoming_schema: str = "serving_incoming"):
    with conn.transaction():
        conn.execute("SET LOCAL lock_timeout = '5s'")
        conn.execute("DROP SCHEMA IF EXISTS serving_previous CASCADE")
        conn.execute("ALTER SCHEMA serving RENAME TO serving_previous")
        conn.execute(f"ALTER SCHEMA {incoming_schema} RENAME TO serving")
```

Caveats worth knowing: the renames take `ACCESS EXCLUSIVE` locks, so a long-running read query will block the swap — hence `lock_timeout`, so you fail fast and retry rather than queueing behind it and blocking every subsequent reader. Also, sessions with prepared statements against the old schema may need to re-plan; in practice, with connections that set `search_path` per checkout, this is fine.

---

# Part 6 — Four levels of orchestration

Your question — "how can every part work automatically when currently everything is manual" — has four possible answers, and the right one is the least sophisticated one that meets your needs.

## L0 — Manual (today)

You run the importer, then the engine, then the build, then restart something. It works. Its failure mode is not that it's slow; it's that steps get skipped or run in the wrong order against stale inputs, and nothing tells you.

## L1 — One command

Before any queue, any worker, any scheduler: **collapse the whole chain into a single command.**

```python
# cli.py
@app.command()
def run_forecast(client: str, revision: int | None = None, publish: bool = True):
    rev = revision or get_latest_published_revision(client)
    snapshot = load_config_snapshot(client, rev)     # Role B
    result   = run_engine(snapshot)
    artifact, manifest = build_artifact(result, rev)
    verify(artifact, manifest)                       # I9 — raises on failure
    if publish:
        publish_artifact(artifact, manifest)
```

This is the highest-value, lowest-effort step in the entire guide. It:

- forces you to make every step programmatic (no notebook cells, no manual file moves);
- makes the pipeline testable end-to-end;
- becomes the body of the worker at L2 and the body of a container job at L3, unchanged.

**Do this first.** Everything else is a question of what calls it.

## L2 — Job table plus worker (recommended)

Publishing a revision inserts a row into a `forecast_runs` table. A long-lived worker polls that table, claims a run, calls the L1 function, and records the outcome. Detailed in Part 7.

Suits you because: no new infrastructure, the audit trail is a table you can query, it survives restarts, and it makes the API responsive (publish returns immediately with a run ID rather than blocking for three minutes).

## L3 — A real orchestrator

Prefect, Dagster, Airflow, or Azure Container Apps Jobs. You get retries, scheduling, dependency graphs, a UI, and backfills without writing them.

**When to move up:** when you have more than roughly five distinct job types with dependencies between them; when you need scheduled backfills across many clients; when someone other than you needs to see run status and re-trigger things; or when a single forecast run needs to fan out across machines.

**Why not now:** every one of those tools is a service to deploy, secure, upgrade, and debug. As a solo developer, an orchestrator outage is an outage you handle alone at 11pm. The 150 lines in Part 7 have no such failure mode.

## L4 — The one that isn't a level: cron

Worth naming explicitly because it's tempting. `cron` triggers on *time*, and your pipeline triggers on an *event* (a revision was published). Time-based triggering means either running constantly and wastefully, or introducing a lag between publish and results appearing.

Cron is genuinely right for: nightly reruns to pick up new market price data, weekly artifact pruning, daily backup verification. It is wrong as the primary trigger for forecast runs.

---

# Part 7 — The job table pattern

## 7.1 Why put the queue in the database

The alternative is Redis plus Celery or RQ. That's a second data store, a second failure mode, a second thing to secure, and an event log that disappears when Redis restarts.

Putting the queue in the database you already have gives you:

- **Transactional enqueue.** Publishing a revision and enqueuing its run happen in one transaction. You cannot end up with a published revision that never got queued, or a queued run for a revision that rolled back. This is the *outbox pattern*, and it's the whole reason the technique exists.
- **A permanent audit trail.** Every run, its inputs, its outcome, its duration, its error — queryable with SQL, forever. This is the same instinct behind your append-only audit log.
- **Zero new infrastructure.**
- **Debuggability.** `SELECT * FROM forecast_runs ORDER BY id DESC LIMIT 20` tells you the state of the world.

The cost: polling is less elegant than a push queue, and this pattern doesn't scale to thousands of jobs per second. You need neither.

## 7.2 Schema

```sql
CREATE TABLE forecast_runs (
    id              BIGSERIAL PRIMARY KEY,

    -- what to run (the idempotency key)
    client_id       TEXT        NOT NULL,
    revision_id     BIGINT      NOT NULL REFERENCES config_revisions(id),
    config_hash     TEXT        NOT NULL,
    engine_version  TEXT        NOT NULL,

    -- lifecycle
    status          TEXT        NOT NULL DEFAULT 'queued'
                    CHECK (status IN ('queued','running','succeeded','failed','cancelled')),
    attempt         INT         NOT NULL DEFAULT 0,
    max_attempts    INT         NOT NULL DEFAULT 3,

    -- scheduling
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    run_after       TIMESTAMPTZ NOT NULL DEFAULT now(),   -- backoff
    claimed_at      TIMESTAMPTZ,
    heartbeat_at    TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,

    -- who
    worker_id       TEXT,
    triggered_by    TEXT        NOT NULL,   -- 'publish' | 'manual' | 'schedule' | 'retry'

    -- results
    artifact_name   TEXT,
    manifest        JSONB,
    error_class     TEXT,
    error_message   TEXT,
    error_traceback TEXT
);

-- Idempotency (I6): one live run per (client, revision, engine version).
CREATE UNIQUE INDEX forecast_runs_dedupe
    ON forecast_runs (client_id, revision_id, engine_version)
    WHERE status IN ('queued','running','succeeded');

-- The claim query's access path.
CREATE INDEX forecast_runs_claimable
    ON forecast_runs (run_after)
    WHERE status = 'queued';
```

The partial unique index is doing real work. Publish revision 8 twice and the second insert fails on the constraint — which you catch and treat as success, because a run already exists. A *failed* run is excluded from the index, so retrying after a failure is allowed.

## 7.3 Enqueue, transactionally

```python
def publish_revision(conn, client_id: str, revision_id: int, actor: str) -> int | None:
    """Publish a config revision and enqueue its forecast run. One transaction."""
    with conn.transaction():
        conn.execute(
            "UPDATE config_revisions SET status='published', published_at=now(), "
            "published_by=%s WHERE id=%s AND status='draft'",
            (actor, revision_id),
        )
        config_hash = compute_config_hash(conn, revision_id)

        try:
            row = conn.execute(
                """INSERT INTO forecast_runs
                     (client_id, revision_id, config_hash, engine_version, triggered_by)
                   VALUES (%s, %s, %s, %s, 'publish')
                   RETURNING id""",
                (client_id, revision_id, config_hash, ENGINE_VERSION),
            ).fetchone()
            return row[0]
        except UniqueViolation:
            return None      # already queued or done — idempotent (I6)
```

Both the publish and the enqueue commit together or not at all. This is the property you cannot get with an external queue without distributed-transaction machinery.

## 7.4 The worker

```python
# worker/main.py
import logging, os, signal, socket, time, traceback

WORKER_ID = f"{socket.gethostname()}:{os.getpid()}"
POLL_INTERVAL = 5
log = logging.getLogger("worker")

CLAIM_SQL = """
UPDATE forecast_runs
   SET status='running', claimed_at=now(), heartbeat_at=now(),
       worker_id=%s, attempt = attempt + 1
 WHERE id = (
     SELECT id FROM forecast_runs
      WHERE status='queued' AND run_after <= now()
      ORDER BY created_at
        FOR UPDATE SKIP LOCKED
      LIMIT 1
 )
RETURNING id, client_id, revision_id, config_hash, engine_version, attempt, max_attempts
"""

class Worker:
    def __init__(self, pool):
        self.pool = pool
        self.stopping = False
        signal.signal(signal.SIGTERM, self._stop)
        signal.signal(signal.SIGINT,  self._stop)

    def _stop(self, *_):
        log.info("shutdown requested; finishing current run")
        self.stopping = True

    def run_forever(self):
        while not self.stopping:
            job = self.claim()
            if job is None:
                time.sleep(POLL_INTERVAL)
                continue
            self.execute(job)

    def claim(self):
        with self.pool.connection() as conn:
            return conn.execute(CLAIM_SQL, (WORKER_ID,)).fetchone()

    def execute(self, job):
        run_id = job["id"]
        log.info("run start", extra={"run_id": run_id, "revision": job["revision_id"]})
        try:
            with heartbeat(self.pool, run_id):
                artifact, manifest = build_forecast(
                    client_id      = job["client_id"],
                    revision_id    = job["revision_id"],
                    engine_version = job["engine_version"],
                    run_id         = run_id,
                )
                verify_or_raise(artifact, manifest)      # I9
                publish(run_id, artifact, manifest)      # Part 5
            self.mark_succeeded(run_id, manifest)

        except Exception as exc:
            self.mark_failed(run_id, exc, job)

    def mark_succeeded(self, run_id, manifest):
        with self.pool.connection() as conn:
            conn.execute(
                "UPDATE forecast_runs SET status='succeeded', finished_at=now(), "
                "artifact_name=%s, manifest=%s WHERE id=%s",
                (manifest["artifact_filename"], Json(manifest), run_id),
            )
        log.info("run succeeded", extra={"run_id": run_id})

    def mark_failed(self, run_id, exc, job):
        retryable = isinstance(exc, TRANSIENT_ERRORS) and job["attempt"] < job["max_attempts"]
        backoff   = min(300, 10 * 2 ** job["attempt"])
        with self.pool.connection() as conn:
            conn.execute(
                """UPDATE forecast_runs
                      SET status = CASE WHEN %s THEN 'queued' ELSE 'failed' END,
                          run_after = now() + make_interval(secs => %s),
                          finished_at = CASE WHEN %s THEN NULL ELSE now() END,
                          error_class=%s, error_message=%s, error_traceback=%s
                    WHERE id=%s""",
                (retryable, backoff, retryable,
                 type(exc).__name__, str(exc)[:2000], traceback.format_exc()[:20000], run_id),
            )
        log.exception("run failed", extra={"run_id": run_id, "will_retry": retryable})
        if not retryable:
            alert(f"the Company run {run_id} failed permanently: {exc}")
```

**`FOR UPDATE SKIP LOCKED`** is the key line. It means "give me a queued row that no other worker has locked, and don't wait for contended ones." It's how Postgres does work queues, and it makes running two workers safe with no coordination.

SQLite has no `SKIP LOCKED`, but with one writer it doesn't need it: wrap the select-then-update in `BEGIN IMMEDIATE`, which takes the write lock up front and serializes claims correctly.

## 7.5 Heartbeats and orphan recovery

If the worker is killed mid-run (OOM, VM reboot, deploy), a row sits in `running` forever and its idempotency key blocks re-enqueue. Fix: heartbeat while working, and reap stale rows.

```python
@contextmanager
def heartbeat(pool, run_id, interval=30):
    stop = threading.Event()
    def beat():
        while not stop.wait(interval):
            with pool.connection() as conn:
                conn.execute("UPDATE forecast_runs SET heartbeat_at=now() WHERE id=%s", (run_id,))
    t = threading.Thread(target=beat, daemon=True); t.start()
    try:
        yield
    finally:
        stop.set()
```

```sql
-- Reaper: run every few minutes (a cron job or a branch of the worker loop).
UPDATE forecast_runs
   SET status      = CASE WHEN attempt < max_attempts THEN 'queued' ELSE 'failed' END,
       error_class = 'WorkerLost',
       error_message = 'no heartbeat for 5 minutes',
       run_after   = now()
 WHERE status = 'running'
   AND heartbeat_at < now() - interval '5 minutes';
```

Set the threshold well above your longest expected run. If a full rebuild takes three minutes, five is fine; if it grows to twenty, raise it or the reaper will start killing healthy runs.

## 7.6 The API surface

Long jobs must not run inside an HTTP request. Load balancers, proxies, and browsers all time out; a three-minute request is a request that fails at two minutes. Return immediately and let the client poll.

```python
@app.post("/api/config/{revision_id}/publish", status_code=202)
def publish_endpoint(revision_id: int, actor: str = Depends(current_user)):
    run_id = publish_revision(conn, client_id, revision_id, actor)
    return {"run_id": run_id, "status": "queued",
            "poll": f"/api/runs/{run_id}"}

@app.get("/api/runs/{run_id}")
def run_status(run_id: int):
    r = fetch_run(run_id)
    return {
        "run_id":   r["id"],
        "status":   r["status"],
        "attempt":  r["attempt"],
        "started":  r["claimed_at"],
        "finished": r["finished_at"],
        "error":    r["error_message"],
        "artifact": r["artifact_name"],
    }
```

The React side polls `/api/runs/{id}` every couple of seconds while status is `queued` or `running`, then refetches the forecast data. Simple, robust, and no WebSocket infrastructure. Move to server-sent events only if the polling ever bothers you.


---

# Part 8 — Failure modes

Design work is largely the enumeration of failure modes. Here are the ones this system actually has, what detects each, and what the design does about it.

| # | Failure | Detected by | Response | Design element that handles it |
|---|---|---|---|---|
| 1 | Engine raises mid-run | Exception in worker | Row → `failed` with traceback; alert | Job table, error columns |
| 2 | Worker process killed mid-run | Missing heartbeat | Reaper re-queues or fails the row | §7.5 |
| 3 | VM reboots | Everything restarts | systemd restarts services; `running` rows reaped | §10.3 |
| 4 | Build succeeds but verification fails | Contract checks | Artifact never published; previous stays live | I9, §5.2 |
| 5 | Build succeeds, verification passes, numbers still wrong | **Nothing, currently** | See §8.1 — this is the dangerous one | Golden-master tests, §14.2 |
| 6 | Two publishes race | Unique index violation | Second is a no-op | §7.2 idempotency index |
| 7 | Artifact written, pointer swap fails | Exception during publish | Old artifact stays `current`; run marked failed | Atomic rename, §5.2 |
| 8 | Disk fills with old artifacts | Disk-space monitoring | Retention job prunes | §4.5 |
| 9 | Long run exceeds HTTP timeout | N/A — doesn't happen | Publishing returns 202 immediately | §7.6 |
| 10 | Config DB unavailable | Connection errors | Serving continues from artifact; publishing blocked | I8 |
| 11 | Corrupt artifact on disk | SHA mismatch at load | Refuse to serve; roll back | Manifest `artifact_sha256` |
| 12 | Engine upgraded, old artifacts incompatible with new API | Contract version check | API refuses; rebuild | §13.2 |
| 13 | Someone edits a published revision | Config hash mismatch | Detected at next run | Manifest `config_content_hash` |
| 14 | Client data loss (config DB) | Nothing until someone looks | **Restore from backup — must be tested** | Part 11 |
| 15 | Certificate expires | Site down, browser errors | Automatic renewal | Caddy, §10.2 |

## 8.1 The one that should worry you most

Failure 5 — a run that completes, passes every check, and produces numbers that are wrong.

This is qualitatively different from the others. Every other failure is loud. This one is silent, ships to a client, gets used in a financing decision, and surfaces weeks later. In a financial forecasting product, this is the failure that damages you.

Your existing checks catch *internal inconsistency*: arithmetic conservation, drill-path reconciliation, FK integrity, grain separation. Those are genuinely good and they catch a real class of bug. What they cannot catch is a change that is internally consistent and externally wrong — a sign flip that propagates uniformly, an off-by-one in a debt amortization schedule, a fiscal-year boundary applied a month early.

Three defences, in increasing order of effort:

**Golden-master tests.** Freeze a representative input configuration. Freeze the engine's output for it. On every commit, rerun and diff. Any change to any number fails the test and must be explicitly acknowledged by updating the golden file in the same commit. This turns "did I change something I didn't mean to?" from a hope into a build failure. For a forecasting engine this is the single highest-value test you can write, and it costs an afternoon.

**Run-over-run diffing.** When a new artifact is built, compare its aggregates against the previous artifact for the same client. Report the deltas. Large unexplained movements in a line item that shouldn't have changed are a signal. You don't have to block on this — surfacing it in the run record and in the UI is most of the value.

**Property assertions from domain knowledge.** Things that must be true of any valid grain-farm forecast, regardless of inputs: operating loan balance never exceeds the credit limit; cumulative cash flow reconciles to opening plus inflows minus outflows; depreciation never exceeds remaining book value; a crop's revenue is zero in months with no harvest. Each one you encode is a bug that can never ship silently again.

## 8.2 The failure-mode habit

When you add a feature, ask: what happens if this fails halfway? Most of the design decisions in this document are answers to that question asked in advance. The build-aside-then-swap pattern exists because "what if the build fails halfway" has a bad answer if you're writing in place.

---

# Part 9 — Azure topologies and cost

Four realistic shapes, cheapest and simplest first. Prices below are on-demand USD, gathered September 2026; treat them as within ±20% and verify in the Azure pricing calculator for **Canada Central**, which is the region you want (Part 12.4).

## 9.1 Option A — Single VM, everything on it

```
┌────────────────────── Azure VM (Linux, B2s-class) ──────────────────────┐
│                                                                          │
│   Caddy  ──▶  FastAPI (systemd)  ──▶  artifacts/current  (SQLite file)   │
│    TLS        static React build                                         │
│                                                                          │
│   Worker (systemd)  ──▶  forecast engine  ──▶  builds artifacts          │
│                                                                          │
│   SQLite config DB  ──▶  Litestream  ──────────────────┐                 │
│   artifacts/  ──▶  nightly sync  ─────────────────────┐│                 │
└───────────────────────────────────────────────────────┼┼─────────────────┘
                                                        ▼▼
                                            Azure Blob Storage (backups)
```

- **Cost:** VM ~$25–35/mo (B2s-class, 2 vCPU / 4 GiB) + managed disk ~$5 + blob storage ~$1–3. **≈ $32–43/mo.**
- **Pros:** one thing to deploy, one thing to debug, one SSH session. No network hops between components. Everything is a file you can look at.
- **Cons:** single point of failure; you own OS patching; backups are your responsibility; no horizontal scaling.
- **Availability:** a single Azure VM has no meaningful uptime guarantee beyond premium-disk SLAs, and reboots for host maintenance happen. For an internal-facing forecasting tool used by a handful of farm clients, this is almost certainly acceptable — but it should be a decision, not an accident.

## 9.2 Option B — VM plus managed Postgres (recommended)

Same VM, but the configuration store moves to Azure Database for PostgreSQL Flexible Server.

- **Cost:** VM ~$30 + disk ~$5 + Postgres Burstable B1ms (1 vCPU / 2 GiB) from about **$12/mo compute** in cheap US regions (a bit more in Canada Central), plus storage and backup billed separately — backup storage is free up to 100% of your provisioned server storage, with additional consumption billed per GiB-month — call it $18–28 all-in for the database. **≈ $55–70/mo.**
- **What the extra ~$25 buys:** automated backups with point-in-time restore, a real type system for your irreplaceable data, RLS available for multi-tenancy, and someone else's job to patch the database.
- Burstable instances also support stop/start, so a non-production server costs only storage while stopped — useful for a staging environment you don't run continuously.
- **Caution on Burstable:** B-series accrues CPU credits and throttles hard when they're exhausted. Burstable compute is designed for workloads that stay idle or below baseline most of the time. Your configuration workload genuinely is that. But if you ever move the *serving* workload onto Postgres, watch credit exhaustion — a throttled database looks exactly like a mysterious slowdown.

## 9.3 Option C — Container Apps plus managed Postgres plus Static Web Apps

API and worker as containers on Azure Container Apps; React on Azure Static Web Apps; Postgres managed.

- **Cost:** Static Web Apps has a free tier intended for hobby/personal use with no SLA (Standard is ~$9/mo and has one). Container Apps gives the first 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests each month free, and bills per-second on allocated vCPU and memory beyond that, with a reduced idle rate for replicas at minimum count that aren't processing requests. Plus Postgres $18–28. **≈ $30–60/mo**, highly variable.
- **Pros:** no OS to patch, scale-to-zero, rolling deploys, containers as the deployment unit.
- **Cons that matter for you:**
  - **The artifact model breaks.** Container filesystems are ephemeral. Artifacts must live in Azure Files (mountable but slow for SQLite, and network filesystems plus SQLite locking is a well-known hazard) or move into Postgres entirely — which means Configuration 3 and losing the file-artifact benefits.
  - **Scale-to-zero fights the worker.** A worker polling a job table must stay alive. You'd need a minimum replica of 1 (paying idle rates) or KEDA-driven scaling on a Postgres query, which is more machinery.
  - **More concepts to hold**: revisions, ingress, environments, registries, managed identities, Dapr if you drift into it. Every one is a thing to debug alone.
- **Verdict:** the right answer eventually, not now. Choose it when you have more than one developer, or when deploy friction on the VM starts costing real time.

## 9.4 Option D — Fully managed everything

App Service for the API, Container Apps Jobs for the engine, Postgres Flexible Server, Static Web Apps, Key Vault, Application Insights, Service Bus.

- **Cost:** $100–200/mo and climbing.
- **Verdict:** this is the architecture you'd draw on a whiteboard to impress someone, and it's the wrong one for a solo developer with one client. Every managed service is a resource with its own identity, networking, diagnostics, and pricing model. The complexity is real and it lands entirely on you.

## 9.5 On free tiers

Azure's free offerings are real but time-boxed and easy to misread. New accounts get 750 hours per month each of B2ts v2, B2pts v2, and B2ats v2 burstable VMs free for 12 months, and the free account includes a Burstable B1MS PostgreSQL Flexible Server with 32 GB storage and 32 GB backup free for 12 months, after which it's billed. Two things to internalize:

1. **The cliff is real.** Month 13 arrives and your bill jumps. Put a calendar reminder at month 10 and a budget alert in Azure now.
2. **Stopped is not free.** Even with a stopped server you're still charged for storage and backup storage. Deleting a resource is the only way to stop paying for it.

Also note the B2ats v2 free-tier VM has 2 vCPUs and only 1 GiB of memory. That is not enough RAM to run a pandas forecast engine comfortably alongside an API. Use free-tier VMs for staging, not for the machine that runs the engine.

## 9.6 Recommendation

**Option B.** One VM you fully understand, plus managed Postgres for the data you cannot recreate.

The reasoning is about *where complexity lands*. You are the only person who will debug this system. Option A's complexity is "I have to run backups myself," which is a known, bounded, scriptable problem. Option C and D's complexity is "why is my container unable to reach the database" at 11pm, which is unbounded and involves reading Azure networking documentation. Buy the managed service for the thing where failure is unrecoverable (client configuration data) and self-manage the thing where failure is recoverable (a rebuildable artifact on a disk).

---

# Part 10 — The recommended stack, concretely

## 10.1 Layout on the VM

```
/opt/the Company/                    # application code (git checkout or release tarball)
    api/                            # FastAPI
    worker/                         # job worker
    engine/                         # forecast engine
    web/dist/                       # built React assets
    .venv/

/etc/the Company/
    the Company.env                  # secrets, mode 0600, root-owned
                                    # the Company_DATABASE_URL=postgresql://...
                                    # the Company_ARTIFACT_DIR=/var/lib/the Company/artifacts

/var/lib/the Company/
    artifacts/
        staging/                    # builds land here first (SAME filesystem — §5.2)
        run-000117-r8-b1e4d02.db
        run-000117-r8-b1e4d02.manifest.json
        current -> run-000117-r8-b1e4d02.db

/var/log/the Company/                # or just journald, which is simpler
```

Run everything as an unprivileged `the Company` user. Nothing here needs root.

## 10.2 Caddy for TLS and static assets

```
forecast.the Company.example {
    encode gzip zstd

    handle /api/* {
        reverse_proxy 127.0.0.1:8000
    }

    handle {
        root * /opt/the Company/web/dist
        try_files {path} /index.html      # SPA fallback for client-side routing
        file_server
    }
}
```

Caddy obtains and renews Let's Encrypt certificates automatically with no cron job and no certbot. For a solo operator this eliminates an entire recurring failure mode (failure 15). nginx works too and you'll find more Stack Overflow answers for it, but you'll be managing certificate renewal yourself.

## 10.3 systemd units

```ini
# /etc/systemd/system/the Company-api.service
[Unit]
Description=the Company API
After=network-online.target

[Service]
User=the Company
WorkingDirectory=/opt/the Company
EnvironmentFile=/etc/the Company/the Company.env
ExecStart=/opt/the Company/.venv/bin/uvicorn api.main:app --host 127.0.0.1 --port 8000
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=3
# hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/lib/the Company

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/the Company-worker.service
[Unit]
Description=the Company forecast worker
After=network-online.target

[Service]
User=the Company
WorkingDirectory=/opt/the Company
EnvironmentFile=/etc/the Company/the Company.env
ExecStart=/opt/the Company/.venv/bin/python -m worker.main
Restart=always
RestartSec=10
TimeoutStopSec=600          # let an in-flight forecast finish on SIGTERM
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

`Restart=always` plus the orphan reaper (§7.5) means a crashed worker recovers without you. `TimeoutStopSec=600` plus the SIGTERM handler in §7.4 means a deploy doesn't kill a run mid-flight.

Note the API binds to `127.0.0.1`, not `0.0.0.0`. Caddy is the only thing listening publicly. Combined with an Azure network security group allowing only ports 80, 443, and SSH (ideally SSH only from your IP, or via Azure Bastion), your attack surface is small.

## 10.4 Deployment

Keep it boring. A tarball and a symlink beats a pipeline you have to maintain.

```bash
#!/usr/bin/env bash
# deploy.sh — run from CI or your laptop
set -euo pipefail
HOST=the Company-prod
SHA=$(git rev-parse --short HEAD)

npm --prefix web run build
tar czf "/tmp/the Company-${SHA}.tar.gz" api worker engine web/dist pyproject.toml uv.lock

scp "/tmp/the Company-${SHA}.tar.gz" "$HOST:/tmp/"
ssh "$HOST" bash -s "$SHA" << 'REMOTE'
set -euo pipefail
SHA="$1"
sudo -u the Company mkdir -p "/opt/the Company-releases/${SHA}"
sudo -u the Company tar xzf "/tmp/the Company-${SHA}.tar.gz" -C "/opt/the Company-releases/${SHA}"
cd "/opt/the Company-releases/${SHA}"
sudo -u the Company uv sync --frozen

sudo -u the Company ln -sfn "/opt/the Company-releases/${SHA}" /opt/the Company.new
sudo -u the Company mv -Tf /opt/the Company.new /opt/the Company     # atomic, same trick as §5.2

sudo /opt/the Company/.venv/bin/alembic upgrade head
sudo systemctl restart the Company-api the Company-worker
sleep 3
curl -fsS localhost:8000/health || { echo "HEALTH CHECK FAILED"; exit 1; }
echo "deployed ${SHA}"
REMOTE
```

Same atomic-symlink pattern as artifact publication — once you see it, it's everywhere. Rolling back a *deploy* is the same move as rolling back an *artifact*: point the symlink at the previous release and restart.

## 10.5 What runs where — summary

| Component | Where | Restart policy | Fails if... |
|---|---|---|---|
| Caddy | VM, ports 80/443 | systemd, always | VM down |
| FastAPI | VM, localhost:8000 | systemd, always | VM down, artifact missing |
| Worker | VM | systemd, always | VM down, config DB unreachable |
| Config store | Azure PostgreSQL Flexible Server | managed | Azure regional outage |
| Artifacts | VM disk `/var/lib/the Company/artifacts` | n/a | disk failure (rebuild from config) |
| React assets | VM disk, served by Caddy | n/a | VM down |
| Backups | Azure Blob Storage | n/a | — |

Read the "fails if" column as your availability story. Note that "config DB unreachable" does **not** take down forecast serving. That's invariant I8 earning its keep.


---

# Part 11 — Backups, secrets, logging, alerting

The unglamorous half. Skipping it is how small systems become unrecoverable ones.

## 11.1 Backups

**The invariant: an untested backup is not a backup.** It's a file you hope about.

What actually needs backing up:

| Data | Recoverable without backup? | Strategy |
|---|---|---|
| Config store (Role A) | **No** | Managed PITR (Postgres) or Litestream (SQLite) |
| Serving artifacts (Role C) | Yes — rebuild | Optional: sync last few to Blob for instant rollback |
| Application code | Yes — git | Push to a remote you don't control |
| Secrets / env config | **No** | Encrypted, offline, documented |
| Infrastructure definition | Only from memory | Terraform/Bicep, or at minimum a written runbook |

That last row is the one people miss. If your VM is destroyed, do you know what you clicked to create it? Even a markdown file listing region, VM size, disk type, NSG rules, DNS records, and Postgres parameters is worth an hour.

**If Postgres:** Flexible Server gives you automated backups and point-in-time restore. Set retention to 14–30 days. Then — and this is the part everyone skips — **do a restore drill.** Restore to a new server, point a staging app at it, confirm the data is right. Once, now, and again after any major change. Time it, and write down how long it took: that number is your RTO.

**If SQLite:** Litestream streams WAL frames to Azure Blob Storage continuously and supports point-in-time restore. Two operational notes:

- It replicates through the SQLite API rather than copying files, so it won't corrupt the database and needs no downtime. Never back up a live SQLite file with `cp` — use Litestream, the backup API, or `VACUUM INTO`.
- **Pin the version and read the issue tracker before upgrading.** Litestream has shipped releases with silent replication failures — a bug reported in early 2026 caused replication to stop entirely with no error logged, frozen metrics being the only symptom. The lesson generalizes: *monitor that your backups are actually advancing*, don't just assume the daemon is working. A daily check that the newest object in your backup container has a timestamp from the last hour is five lines and catches this whole class of problem.

**Define your targets explicitly:**
- **RPO** (how much data you can afford to lose): for client configuration, aim for minutes. Litestream and Postgres PITR both get you there.
- **RTO** (how long recovery takes): measure it in a drill; don't estimate it.

## 11.2 Secrets

Rules, in order of importance:

1. **Never in git.** Add a `gitleaks` or `detect-secrets` pre-commit hook. Retroactively removing a committed secret is painful and you must rotate it anyway.
2. **Environment variables via `EnvironmentFile`**, mode 0600, root-owned. Your `pydantic-settings` `Settings` class with the `the Company_` prefix already handles this cleanly — that decision is paying off here.
3. **Azure Key Vault when it earns its place.** For a single VM, an `EnvironmentFile` is defensible. Key Vault matters when you have multiple environments, rotation requirements, or someone auditing you. The intermediate step worth taking early: give the VM a **managed identity** and let it authenticate to Azure services without a stored credential at all.
4. **Rotate what leaks.** Assume any secret that ever touched a laptop, a Slack message, or a screenshot is compromised.

Secrets in this system: Postgres connection string, blob storage credentials (or managed identity), any client-facing auth signing key, SSH keys, and TLS private keys (Caddy manages these itself — don't touch them).

## 11.3 Logging

Switch to **structured logging** now, while the volume is small and the change is cheap.

```python
import logging, json, sys

class JsonFormatter(logging.Formatter):
    def format(self, record):
        payload = {
            "ts":     self.formatTime(record, "%Y-%m-%dT%H:%M:%S%z"),
            "level":  record.levelname,
            "logger": record.name,
            "msg":    record.getMessage(),
        }
        for key in ("run_id", "revision_id", "client_id", "request_id", "artifact"):
            if (val := getattr(record, key, None)) is not None:
                payload[key] = val
        if record.exc_info:
            payload["exc"] = self.formatException(record.exc_info)
        return json.dumps(payload)
```

The critical idea is a **correlation key**. Every log line emitted during a forecast run carries the same `run_id`. Every log line during an HTTP request carries the same `request_id`. Then "what happened during run 117" is one grep, not archaeology across three services.

`journalctl -u the Company-worker -o cat | jq 'select(.run_id == 117)'` gets you a long way before you need a log aggregation service. Note that if you do adopt Azure Monitor / Log Analytics, ingestion is metered and platform logs accumulate silently — set a daily cap.

## 11.4 Alerting

The minimum viable set, in priority order:

1. **A forecast run failed permanently.** Email or SMS from the worker's `mark_failed`. This is the one that matters most — a silently failed pipeline means a client is looking at stale numbers without knowing it.
2. **Backups have stopped advancing.** Daily check on the newest backup object's age.
3. **The site is down.** An external uptime checker hitting `/health` every few minutes. Several are free. Critically, this runs *outside* your VM, so it still works when your VM doesn't.
4. **Disk above 80%.** Artifacts accumulate.
5. **Azure budget alert.** Catches both the free-tier cliff and a runaway resource.

Make `/health` meaningful, not just `return {"ok": True}`:

```python
@app.get("/health")
def health(request: Request):
    art = request.app.state.artifacts.current
    age_hours = (utcnow() - parse(art.manifest["build"]["finished_at"])).total_seconds() / 3600
    return {
        "status":        "ok",
        "artifact":      art.path.name,
        "run_id":        art.manifest["run_id"],
        "revision_id":   art.manifest["inputs"]["config_revision_id"],
        "artifact_age_hours": round(age_hours, 1),
        "contract_version":   art.manifest["contract_version"],
    }
```

`artifact_age_hours` is the sneaky-useful one: it catches "everything is up but nothing has run in a week," which no process-liveness check will ever notice.

---

# Part 12 — Security and multi-tenancy

This is the section with the highest ratio of future pain to present effort.

## 12.1 The client #2 question

Right now you have one client's workbook imported. The design decision you make *before* client #2 determines whether onboarding them takes a day or a month. Three models:

**Model 1 — Shared database, `client_id` on every table.**
One config database, one artifact, every query filtered by `client_id`.
- *Pro:* one of everything; cross-client analytics are trivial.
- *Con:* every single query must filter correctly, forever. One missed `WHERE client_id = ?` leaks one farm's financials to another. In a product handling farm debt schedules and cash positions, that's not a bug, it's an incident with legal weight.
- *Mitigation:* Postgres **row-level security** enforces the filter in the database rather than in your application code, so a forgotten `WHERE` clause fails closed. This is a strong reason to have Postgres for Role A.

**Model 2 — Shared config database, artifact per client.**
Config in one Postgres database with `client_id` and RLS; each forecast run produces `current/{client_id}.db`.
- *Pro:* serving-side isolation is physical — a query against Pederson's artifact literally cannot return Johnson's rows, because those bytes aren't in the file. Rebuilds are per-client, so one client's bad config doesn't block another's forecast. Runs parallelize naturally.
- *Con:* N artifacts to manage; cross-client reporting requires opening several files (DuckDB is good at this via `ATTACH`).
- **This is the recommended model**, because it falls out of the artifact design for free and gives the strongest isolation exactly where the risk is highest.

**Model 3 — Full stack per client.**
Separate database, separate VM, separate everything.
- *Pro:* maximum isolation; a per-client compliance story that sells itself to a cautious lender.
- *Con:* cost and operational load multiply linearly. Untenable solo beyond a handful of clients.

**Recommendation: Model 2, and structure for it now.** Concretely: put `client_id` in the config schema and in the artifact filename today, even with one client. Retrofitting a tenant key into a schema that has production data in it is one of the more miserable refactors in software.

## 12.2 Authentication and authorization

Questions to answer before you have users:

- **Who logs in?** You only? Farm owners? Their lender or accountant? Each answer implies a different model.
- **Do clients see raw configuration or only forecasts?** Different permission surfaces entirely.
- **Can a client edit their own inputs?** If yes, your config store gains concurrent writers and an audit requirement that isn't optional.

For authentication itself: **don't build it.** Password reset flows, session invalidation, MFA, and account recovery are a lot of surface area to get right alone. Use Microsoft Entra ID (natural on Azure), or Auth0/Clerk/Supabase Auth. Your API validates a JWT; you write no password code.

**The authorization invariant:** every request's tenant scope is derived server-side from the verified token, never from a request parameter. If your API accepts `?client_id=` from the frontend and trusts it, you have an authorization bug regardless of what the UI sends. This is the most common serious vulnerability in exactly this kind of application.

## 12.3 Network posture

- Azure NSG: inbound 443 and 80 only, from anywhere; SSH restricted to your IP or via Azure Bastion.
- Postgres Flexible Server: private access (VNet integration), or public access with a firewall rule for the VM's IP only. Never open to `0.0.0.0/0`, and require TLS (`sslmode=require` at minimum).
- API bound to `127.0.0.1`, never `0.0.0.0`.
- No database ports exposed publicly. Ever. Automated scanners find open Postgres in minutes.

## 12.4 Data residency and regulation

Canadian grain farms' financial data, likely used in lending decisions.

- **Deploy in Canada Central or Canada East.** Not because it's certainly legally required, but because the first sophisticated client — or their lender — will ask where the data lives, and "Canada" is a much better answer than "Virginia." Region cannot be changed after resource creation; you'd rebuild. Choosing correctly on day one costs nothing.
- **PIPEDA** applies to personal information in commercial activity. Farm business financials may include personal information (sole proprietorships, personal guarantees, individual identifiers). Worth an hour with an actual lawyer before your first paying client, not after.
- **Encryption at rest** is on by default for Azure managed disks and Postgres. Confirm it and note it — it's a checkbox on every vendor security questionnaire you'll eventually receive.
- **Retention and deletion.** What happens to a client's data when they leave? Having an answer before you're asked is much easier than inventing one under pressure.
- **The audit log you already built** is a real asset here. Who changed which assumption, when, is exactly what a lender or a dispute will ask about.

---

# Part 13 — Migrations and contract versioning

## 13.1 Config store migrations

Once real client data exists, schema changes stop being "drop and recreate." Adopt **Alembic** now, before that's true.

- Every schema change is a versioned, reviewed migration file in git.
- Migrations run as a deploy step (see §10.4), before services restart.
- **Test the downgrade path** at least for the migrations you'd plausibly need to reverse. Many aren't reversible (dropped columns lose data) — know which are.
- **Expand/contract for anything risky:** add the new column, backfill it, switch the code to read it, and only then drop the old one, across separate deploys. Never rename a column in one step in a system you can't take down.
- Alembic works with SQLite too, but SQLite's limited `ALTER TABLE` forces table-rebuild migrations (`batch_alter_table`), which are slower and fussier. Another quiet point for Postgres in Role A.

## 13.2 Artifact contract versioning

The serving artifact needs no migrations — it's rebuilt. But the **contract between artifact and API** does need versioning, because deploys aren't instantaneous and rollbacks happen.

The scenario: you change the artifact schema, deploy a new API and worker together, and then need to roll back the API. Now a new-format artifact is being read by an old API, which doesn't understand it.

```python
# api/contract.py
SUPPORTED_CONTRACT_VERSIONS = {3, 4}    # this API build can read either
```

The `ArtifactHolder` in §4.4 refuses to load an artifact outside that set. That converts a subtle data-shape bug into a loud startup failure — which is exactly the trade you want.

The rule for changes: **additive changes don't bump the version** (a new column an old API ignores is harmless). **Removals and semantic changes do.** Widen `SUPPORTED_CONTRACT_VERSIONS` in one deploy, and only remove the old version from the set after every live artifact has been rebuilt.

## 13.3 Frontend/API skew

Your React bundle is cached in browsers. After a deploy, a user may hold an old bundle calling a new API — or vice versa.

- Version the API path (`/api/v1/...`) when you make breaking changes.
- Have the frontend send its build SHA in a header; the API can log skew, or return a "please refresh" signal when it's too old.
- Practically: keep the API backward-compatible for one release cycle and this rarely bites. But know it exists, because the symptom ("it works for me but one user sees an error") is otherwise baffling.

---

# Part 14 — Environments and testing

## 14.1 Environments

Three, all running identical code with different configuration:

| Environment | Where | Data | Purpose |
|---|---|---|---|
| Local | Your machine | Synthetic or anonymized | Development |
| Staging | Small VM, or the prod VM with a second systemd unit + separate directories | Copy of prod config, refreshed periodically | Test deploys, migrations, restore drills |
| Production | The VM | Real client data | Serving |

Your `pydantic-settings` `Settings` class with URI-based location resolution already makes this a matter of which `EnvironmentFile` gets loaded. That was a good decision and this is where it pays.

**The rule: never test a migration or a restore in production first.** Staging exists specifically so the first time you run `alembic upgrade head` against real-shaped data, it isn't against real data.

Cost note: staging can be a stopped Burstable Postgres server (storage-only cost) and a free-tier VM, spun up only when you're deploying.

## 14.2 The testing pyramid for a forecast engine

**Unit tests** for individual calculations. Amortization schedules, proration, fiscal-year boundaries. Fast, numerous, and where you encode the domain rules you keep re-deriving.

**Golden-master tests** — the highest-value tier for you, as argued in §8.1. Freeze inputs, freeze outputs, diff on every commit. Concretely:

```python
def test_golden_master_pederson(tmp_path):
    snapshot = load_fixture("fixtures/pederson_r8_config.json")
    result   = run_engine(snapshot, as_of=date(2026, 9, 1))
    actual   = normalize(result)                       # sort, round to cents
    expected = json.loads(Path("fixtures/pederson_r8_expected.json").read_text())
    assert actual == expected, "engine output changed — review and update the golden file"
```

When the test fails legitimately (you fixed a bug, the numbers should change), you regenerate the golden file *in the same commit* as the code change. The diff in code review then shows exactly which numbers moved and by how much. That artifact — a reviewable diff of financial outputs — is worth more than the test itself.

**Contract tests** — your existing verification suite, run as a build gate rather than manually.

**Integration tests** — publish a revision, assert a run is enqueued, run the worker, assert an artifact appears and `current` points at it. One or two of these catch whole categories of wiring bugs.

**Determinism test** — run the same input twice, assert identical output. Cheap, and directly enforces invariant I2. Catches unordered-dict iteration, unseeded randomness, and wall-clock leakage into the engine.

## 14.3 CI

GitHub Actions, one workflow: on push, install, lint, run tests. On a tag or on main, additionally build the frontend and run `deploy.sh`. Don't build a sophisticated pipeline — build one that runs the tests, because the value is the tests running, not the pipeline.


---

# Part 15 — Questions you didn't know to ask

Grouped by category. Each is a question, why it matters, and whether it needs answering now or can wait.

## 15.1 Correctness and trust

**Can you reproduce a forecast you produced three months ago?**
A client will ask why a number changed between two conversations. Without pinned engine versions and immutable revisions, you cannot answer. *Now* — it's an architectural property, not a feature you bolt on.

**When numbers change between runs, can you explain why?**
Run-over-run diffing (§8.1). *Soon.* This is also a genuinely good product feature, not just an internal tool.

**What's your tolerance for a wrong number reaching a client?**
The answer shapes how much you invest in §8.1's defences. For a tool informing borrowing decisions, the tolerance should be near zero. *Now.*

**Are you using floats for money anywhere?**
*Now.* Cheap to fix before client data, expensive after.

**What is "as of" for a forecast?**
Does a forecast run on Sept 10 against revision 8 assume today's date, the revision's publish date, or a fiscal-year start? Ambiguity here produces bugs that are almost impossible to spot because the output always looks plausible. Make `as_of` an explicit engine parameter, record it in the manifest. *Now.*

**Time zones.** Saskatchewan does not observe daylight saving; your future Ontario base does. If any date logic uses local time, that difference will eventually produce an off-by-one-day. Store everything in UTC, convert at the presentation edge, and make the fiscal calendar explicit rather than derived from a timestamp. *Now* — cheap to get right, nasty to retrofit.

## 15.2 Product and client-facing

**What does a client do when they think a number is wrong?**
Is there a support path? Can you reproduce their exact view? Displaying the build stamp (I3) makes this tractable. *Soon.*

**Can a client export?**
Farm operators and their lenders will want Excel. Your engine's data is already tabular. An export endpoint is a day of work and removes a major objection during sales conversations. *Soon.*

**Do clients need historical forecasts, or only the current one?**
If they need to compare "what we projected in March" against "what we project now," your artifacts are already the answer — you just need to keep more of them and expose selection. This is a real product differentiator that your architecture gives you almost free. *Worth deciding early*, because it affects retention policy.

**What happens when a forecast is running and a client is viewing?**
Answered: they see the old artifact until the swap, then the new one. Worth confirming the UI doesn't do something jarring mid-session.

**How do clients get onboarded?**
Currently: you import their Excel workbook by hand. Is that the permanent model? A one-hour manual onboarding is fine at 5 clients and a bottleneck at 50. *Later*, but know which you're building toward.

## 15.3 Operational

**If the VM is destroyed right now, what's your recovery time?**
If you don't know, that's the answer: unknown. *Now* — write the runbook, do the drill.

**Who gets notified when something fails at 2am?**
You. Is there a path to reach you? Do you want to be reachable? A failed nightly run that waits until morning is probably fine; decide that deliberately rather than discovering it.

**What's the disk growth rate?**
Artifacts × clients × runs per day × retention. Calculate it once. Disk-full is a boring outage that takes down everything at once.

**What's your Azure spending limit?**
Set a budget alert. Cloud bills surprise people in exactly one direction.

**How do you get a shell when SSH breaks?**
Azure Serial Console exists. Know it exists before you need it.

**Is anything in this system dependent on your laptop?**
A script only on your machine, a credential only in your keychain, a step only in your head. Every one is a single point of failure with no redundancy.

## 15.4 Architectural

**What happens when a forecast run takes 30 minutes instead of 3?**
Complexity grows. Does your heartbeat threshold still work? Does the UI still poll sensibly? Does the worker still fit in the VM's memory? *Watch the trend* — record duration in every run row so you have the data.

**What if two clients need forecasts simultaneously?**
With one worker, they serialize. With per-client artifacts (§12.1) you can run two workers safely — `SKIP LOCKED` already handles the coordination. *Later*, and cheap because you designed for it.

**Does the engine fit in memory?**
pandas is memory-hungry. If a client's data is 20× larger, does a B2s VM survive it? Memory limits on a VM present as the OOM killer terminating your worker mid-run — which looks like failure 2, not like a resource problem. Log peak memory per run so the eventual diagnosis takes minutes.

**Should the frontend be on the same host?**
It works now. Moving to a CDN or Static Web Apps later is a small change with the Caddy setup, since it's just a static bundle.

**Are you accumulating a monolith or a set of components?**
The engine, worker, and API share the codebase. That's correct at this size. The boundary that matters is *the data contract*, not the process boundary — and you've already specified that formally, which is the hard part.

## 15.5 Personal and business

**What's the bus factor?**
One. That's fine and normal for this stage, but it means documentation isn't optional overhead — it's the only continuity mechanism that exists. This document is part of that. So is the runbook.

**What are you optimizing for over the next six months?**
Shipping to the first paying client, or building for ten? They imply different answers throughout this document. The recommendations here assume the former, with the latter kept cheap to reach.

**Which decisions are one-way doors?**
Most of what's here is reversible. The genuine one-way (or expensive-to-reverse) doors: Azure region, tenant model, whether money is float or decimal, and whether you have real backups when you need them. Spend your care there and move fast on the rest.

---

# Part 16 — Sequencing

An ordered plan. Each phase has an exit criterion — something demonstrably true, not "it feels done."

**Phase 1 — Make it one command.** (Highest value per hour in the whole plan.)
- Collapse import → engine → build → verify → publish into a single CLI entry point (§6, L1).
- Build into `staging/`, verify, then atomic-swap into place (§5.2).
- Write the manifest (§4.3) and surface the build stamp in every API response (I3).
- Write `rollback.sh` and run it once.
- *Exit:* one command produces a verified, published artifact; rollback demonstrably works.

**Phase 2 — Make it correct.**
- Convert money to `Decimal`/`NUMERIC` if it isn't already.
- Golden-master test for one client (§14.2).
- Determinism test.
- Contract verification as a hard gate (I9).
- Explicit `as_of` parameter; UTC everywhere.
- *Exit:* a code change that alters any output number fails CI until the golden file is updated.

**Phase 3 — Get it hosted.**
- Provision Canada Central: VM, NSG, DNS, Caddy, systemd units.
- Write the infrastructure runbook as you go — you'll never have better recall than during the build.
- Deploy script (§10.4).
- External uptime check, Azure budget alert.
- *Exit:* the system runs unattended on Azure and survives a `reboot`.

**Phase 4 — Make it automatic.**
- `forecast_runs` table, transactional enqueue on publish, worker with claim/heartbeat/retry (Part 7).
- `POST /publish` returns 202; frontend polls run status.
- Orphan reaper.
- Alert on permanent failure.
- *Exit:* publishing a revision in the UI results in updated forecasts with no terminal involvement.

**Phase 5 — Make it survivable.**
- Migrate the config store to Postgres if going with Configuration 2, with Alembic from the first migration.
- Backups configured; **restore drill performed and timed**.
- Structured logging with `run_id` correlation.
- Backup-freshness check.
- *Exit:* you have personally restored the config store from backup into a working staging environment, and you know how long it took.

**Phase 6 — Make it multi-client.**
- `client_id` through the config schema, artifact names, and API scope.
- Real authentication; tenant scope derived from the verified token, never a request parameter.
- Row-level security if on Postgres.
- *Exit:* two clients' data coexist and neither can observe the other, verified by an actual test.

Phases 1 and 2 are worth doing before anything else regardless of your deployment timeline, because they make every subsequent phase safer. Phase 3 can move earlier if you have an external deadline — hosting a manually-triggered system is legitimate, as long as you know that's what you're doing.

---

# Appendix A — Cost reference

USD per month, on-demand, approximate as of September 2026. Verify in the Azure pricing calculator for Canada Central before committing.

| Component | Option | Est. cost |
|---|---|---|
| VM | B2s / B2s_v2 class (2 vCPU, 4–8 GiB) | $30–60 |
| VM | B2ats_v2 (2 vCPU, 1 GiB) — staging only | ~$5 |
| Managed disk | 64 GB Premium SSD | ~$5–10 |
| PostgreSQL | Flexible Server Burstable B1ms, compute only | from ~$12 |
| PostgreSQL | + 32 GB storage and backups | ~$18–28 all-in |
| Blob Storage | Backups, tens of GB | $1–3 |
| Static Web Apps | Free tier (no SLA) / Standard | $0 / ~$9 |
| Container Apps | Within monthly free grant | $0 |
| Public IP | Static | ~$4 |
| **Configuration 1** (all-SQLite, one VM) | | **~$35–45** |
| **Configuration 2** (VM + managed Postgres) | | **~$55–75** |

Cost-control notes: the 12-month free tier expires (§9.5); stopped resources still bill for storage; reserved instances cut compute 30–40% for a 1-year commitment, which becomes worth it once you're confident in the shape; and egress is billed but negligible at your volume.

---

# Appendix B — Decision triggers

When to revisit each choice. Written so future-you can check a condition rather than re-reason from scratch.

| Decision | Current choice | Revisit when |
|---|---|---|
| Config store | Postgres Flexible Server | Never for scale; possibly for cost if pre-revenue |
| Serving store | SQLite artifacts | A drill-down endpoint exceeds ~200ms, or you need >1 API host → DuckDB or Postgres schema-swap |
| Orchestration | Job table + worker | >5 job types with dependencies, or backfills across many clients → Prefect/Dagster |
| Compute | Single VM | Deploy friction costs real hours, or you need HA → Container Apps |
| Frontend hosting | Caddy on the VM | Clients outside the region complain about latency → Static Web Apps / CDN |
| Tenancy | Artifact per client, shared config w/ RLS | A client contractually requires physical isolation → dedicated stack |
| Auth | Hosted provider | Never build your own |
| Queue | Postgres table | >100 jobs/minute (you will not reach this) |
| Backups | Managed PITR / Litestream | After every restore drill that goes badly |
| Money type | Decimal | Never revisit; never use float |
| Region | Canada Central | Cannot change without rebuilding — get it right now |

---

# Appendix C — Glossary

**Artifact** — an immutable build output. Here, one serving database file plus its manifest.

**Atomic rename** — `rename(2)`, which on POSIX either fully replaces the destination or doesn't happen. The basis of every safe swap in this document.

**Connection pool** — a fixed set of pre-opened database connections leased to requests and returned, avoiding per-request connection setup.

**Contract version** — an integer stamped in the manifest describing the artifact's shape, so the API can refuse artifacts it doesn't understand.

**Expand/contract** — a migration technique: add new, backfill, switch reads, then remove old, across separate deploys.

**Golden master** — a frozen expected output, compared against on every run, so unintended numeric changes fail the build.

**Idempotent** — running it twice has the same effect as running it once.

**Manifest** — metadata describing an artifact's inputs, engine version, verification results, and shape.

**Outbox pattern** — enqueuing work in the same transaction as the state change that caused it, so the two can't diverge.

**PITR** — point-in-time recovery: restoring a database to an arbitrary moment, not just the last backup.

**RLS** — row-level security: Postgres enforcing per-row visibility rules in the database, so a missing `WHERE` clause fails closed.

**RPO / RTO** — recovery point objective (how much data you can lose) and recovery time objective (how long recovery takes).

**`SKIP LOCKED`** — a Postgres clause letting a worker claim an unlocked queued row without waiting on rows other workers hold.

**WAL** — write-ahead log. In SQLite, a mode allowing one writer concurrent with many readers; also what Litestream streams to replicate.

---

*Written September 2026. Prices and version-specific notes will drift; the invariants in Part 2 should not.*