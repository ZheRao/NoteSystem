# Local: `os.environ.get` vs. `path_config = read_configs(...)` 

Your two approaches feel like alternatives, but they're actually solving different problems, and that's the key to deciding between them.

```py
INPUT_DIR = Path(
    os.environ.get("GROWLYTICS_INPUT_DIR", PROJECT_ROOT / "data" / "warehouse_exports")
)
```

An env var (`os.environ.get`) is fundamentally an *override mechanism*. Its whole reason to exist is letting a value differ per machine, per deployment, or per CI run without touching code or committed files. A JSON config file is a *structure mechanism* — it lets you describe a bunch of related values (a root plus several subpaths) in one readable place. So the real question isn't "which one," it's "structured defaults that live *where*, overridden *how*."

**The biggest issue is independent of format.** Your JSON is loaded via `files()`, which means it's bundled *inside* the installed package as package data. So despite being called "external config," it's actually internal — baked into the package, and hard to override without editing installed files. And it currently stores `/mnt/c/Users/ZheRao/OneDrive - GROWLYTICS/Database`, which is one specific machine's WSL+OneDrive path. The moment anyone else runs this, or it runs in CI or on a server, that path is wrong, and the only fix is editing a file inside the package. Your `"win"` key suggests you're planning to handle cross-platform by committing each platform's absolute path, which multiplies the problem.

That's the thing to fix regardless of which style you land on: machine-specific absolute paths shouldn't be committed package data. And avoiding that is exactly what env vars are *good* at and your current JSON is *bad* at.

Where each genuinely wins:

Env vars are standard (everyone knows them), need zero parsing code, make per-environment override trivial, and play well with Docker/systemd/CI/`.env` files. They're weak at structure — a flat string namespace — and if you sprinkle `os.environ.get` across modules, your config becomes implicit and hard to discover (no single file lists every knob).

JSON config is structured, hierarchical, and self-documenting — one file shows every path, a single source of truth. It's weak at overrides (especially bundled the way yours is), has no comments, can't compute one path from another (note you already work around this in Python with `Path(root) / user_input`), and saddles you with a custom loader to maintain and test.

The mature pattern is to **layer them rather than choose**: a committed config describes the structure and relative layout (the `system/inputs`-style subpaths), while the machine-specific root comes from an env var (or a gitignored local file, or a path passed at runtime).

Since you flagged wanting schema safeguards "later," `pydantic-settings` collapses all of this into one object and is worth a look:

```python
from pydantic_settings import BaseSettings
from pathlib import Path

class Paths(BaseSettings, env_prefix="GROWLYTICS_"):
    root: Path                       # required; from GROWLYTICS_ROOT env var
    user_input: Path = "UserInputs"  # relative defaults, committed safely
    output: Path = "system/outputs"

    def resolved(self, field: str) -> Path:
        return self.root / getattr(self, field)
```

That gives you nested/typed fields, env-var override, optional `.env`/JSON loading, and validation for free — and it lets you delete `read_configs`.

So, a concrete steer: keep your instinct for structured config, it's good. But get the absolute root out of committed JSON. The smallest change that fixes the real problem is to read just the root from an env var while your JSON holds only the relative subpaths. The cleaner change is to move to `pydantic-settings`.

One small note on your first snippet: the `get()` default is a `Path` while a set env var returns a `str`, so the two branches have different types until `Path()` wraps them — harmless here, but worth knowing.

# Path in Production Env

Great instinct to slow down and build the mental model now, because the single most useful thing here is realizing that the word "path" is hiding three different concepts. Once you separate them, most of the confusion dissolves and the elegant patterns become obvious.

## The three kinds of "path"

**1. Filesystem paths to your data** — where your inputs and outputs actually live (`/mnt/c/Users/.../Database`, a warehouse export folder, an output dir). These are *runtime, environment-specific* locations. They change when you move machines or move to the cloud.

**2. The import path** — how Python finds your *code* when you write `import growlytics_platform`. This is `sys.path` and module resolution. It's what "packaging" is mostly about.

**3. Package-data / resource paths** — files that ship *inside* your package and travel with your code (a bundled default config, a SQL file, a small reference table). These are reached with `importlib.resources.files()`.

Here's the key diagnosis of your earlier code: you discovered tool #3 (`files()`) — which is genuinely the *correct* modern way to locate files bundled with a package — but you pointed it at category #1 data (a machine-specific filesystem location). The mechanism is right; the *kind of thing you stored in it* is wrong. Code-location and data-location need to be separated. That separation is the whole lesson.

## Why packaging changes everything

In local development you almost always run `python something.py` from your project root. So `Path("data/foo.csv")` "works" — but only because your current working directory (CWD) happens to be the root. It's working by accident of *where you launched Python from*.

The moment your code is packaged and `pip install`-ed, that accident disappears:

- Your code can land anywhere — a virtualenv's `site-packages`, a Docker image's `/usr/local/lib/...`, a Lambda layer, even inside a zip. You no longer know where your own code physically sits.
- CWD is no longer reliably the project root; someone might run your tool from any directory.
- So CWD-relative paths, and `Path(__file__).parents[N]` tricks, and a computed `PROJECT_ROOT` all become unreliable.

That last one matters for you specifically: your first snippet had a `PROJECT_ROOT`. Basing data locations off `__file__` is fine for a script you run in place, but it doesn't survive packaging — after install there *is* no project root, only installed modules, and the `parents[N]` count is brittle (move one file and it breaks).

`importlib.resources` exists precisely to solve this: it finds files that ship *with* your code regardless of where it got installed. Use it for #3. Never use it for #1.

The deep shift: **in a packaged world, "where am I?" stops being answerable from filesystem position. You get your bearings from explicit configuration that's handed to you, not from implicit position (CWD, `__file__`).**

## The elegant pattern: layered config

The mature solution is a precedence stack, highest wins:

1. Command-line args (per-invocation)
2. Environment variables (per-deployment / per-machine)
3. A config file (structured, environment-level)
4. Defaults baked into code (sensible fallbacks, relative structure)

Data *structure* (your relative subpaths like `system/inputs`) belongs in layers 3–4 and can safely live in the package. The *machine-specific root* belongs in layers 1–2 and must never be committed. You collapse all of this into **one Settings object** that the rest of your code reads from, so config is defined in exactly one discoverable place. `pydantic-settings` implements this stack for you and gives you the schema validation you said you wanted later.

## What cloud migration does to this

This is the part worth front-loading, because your plan ("modify the external config after migration") is exactly the thing that breaks:

- In the cloud you usually don't have local filesystem paths at all — storage is object storage accessed by URI (`s3://bucket/key`, `gs://...`), not POSIX paths.
- You don't edit files inside a built container image; the image is immutable. Config is *injected at deploy time*, almost always through environment variables. That's *why* **env-var-driven** config is the cloud norm.

So the cloud forces config to be external and injected — which is the same direction the layered pattern was already pushing you. If you want locations to work both locally and in the cloud, you eventually treat a "location" as a URI rather than a hardcoded `Path`; libraries like `cloudpathlib` or `universal_pathlib`/`fsspec` let `s3://...` behave like a `Path`. You don't need that yet, but it's the horizon.

## Recommended evolution path

You can absolutely ship a hacky prototype first — that's a legitimate stage, not a sin. Here's the staircase:

- **Stage 0 (now):** local, CWD-relative, hardcoded JSON. Fine to prototype.
- **Stage 1:** separate code-location from data-location. Keep `files()` for package data, but pull the absolute root out of committed JSON and read it from an env var. The JSON keeps only relative structure.
- **Stage 2:** introduce one `pydantic-settings` Settings object as the single source of config, with the layered precedence. Delete your custom `read_configs`.
- **Stage 3 (cloud):** make locations URI-aware and inject all config via env vars at deploy.

Two practical things to learn next, in order of leverage: first, **editable installs** — run `pip install -e .` so you import your code as a real package (`import growlytics_platform`) while still editing it locally. This surfaces packaging path assumptions *early*, on your own machine, instead of at deploy time. Second, skim the modern `importlib.resources` API (`files().joinpath().read_text()`) — which you're already using correctly — so you're confident about what belongs in there versus what doesn't.

If it'd help, I can show a concrete before/after: your current layout and reader rewritten as a small Settings object with the root coming from an env var and the structure staying bundled. Just say the word.

# Cloud Storage Fundamentals

You've already diagnosed the core thing correctly, which is the hard part: a config swap alone isn't enough, because `pathlib.Path` doesn't just assume POSIX *strings* — it assumes a local filesystem with all its *semantics*. So the migration is two halves: address resources by URI (the config part you imagined) **and** route I/O through a layer that understands those URIs (the part you correctly sensed was missing). The good news is your instinct was 80% right; you just need the abstraction underneath it. Here's the whole picture.

## Why a config swap alone isn't enough: object storage isn't a filesystem

Local dev gives you a hierarchical POSIX filesystem: real directories, `open()`, seeking, partial reads, in-place edits, atomic renames, cheap `os.listdir`, OS-level permissions. Cloud "storage" for data systems is almost always **object storage** (S3, GCS, Azure Blob), which is a different beast:

- It's a **flat key-value store**: a bucket plus a key. In `s3://my-bucket/system/inputs/file.csv`, the key is literally `system/inputs/file.csv`. The slashes are *convention* — there are no real folders, just key prefixes.
- Every operation is an **HTTP request over the network** (GET/PUT/LIST/DELETE), with real latency and transient failures — nothing like a local syscall.
- **No in-place modification and no real append.** You PUT a whole object. To "change" a file you rewrite it.
- **No atomic rename/move.** Rename = copy + delete. Moving a million objects is a million copies.
- **No POSIX permissions** — access is via credentials/IAM.
- **Listing is a paginated network call**, not free.

So `Path("/system/inputs") / "file.csv"` is meaningless against S3 not because of the slashes but because the *operations behind it* (open, seek, append, rename, list) don't map. That's why you have to swap the I/O components, exactly as you guessed.

## URIs: the unifier that makes your config dream real

The elegant bridge is the **URI**: `scheme://authority/path`.

```
file:///home/zhe/Database/system/inputs/file.csv   # local
s3://growlytics-data/system/inputs/file.csv         # AWS
gs://growlytics-data/system/inputs/file.csv         # GCP
az://growlytics-data/system/inputs/file.csv         # Azure
```

The *scheme* (`file`, `s3`, `gs`) tells your I/O layer which backend to use. This is the key: if your config stores a **root URI** and your I/O layer dispatches on the scheme, then "just change the config" genuinely works — local dev points the root at `file:///...`, prod points it at `s3://...`, and the same code runs. Your external-config idea was right; it only failed because `pathlib` can't dispatch on scheme. Give it a layer that can, and the dream comes true.

## Two ways to make your I/O URI-aware

**Option 1 — fsspec (a "filesystem" mental model).** `fsspec` presents one uniform Python filesystem interface over many backends (local, `s3fs`, `gcsfs`, `adlfs`, http, memory). You open a URI and get a file-like object, regardless of backend:

```python
import fsspec

# identical code for local and cloud — the scheme picks the backend
with fsspec.open("s3://growlytics-data/system/inputs/file.csv", "rb") as f:
    data = f.read()

with fsspec.open("file:///home/zhe/Database/system/inputs/file.csv", "rb") as f:
    data = f.read()

fs = fsspec.filesystem("s3")
fs.exists("s3://growlytics-data/system/inputs/")   # network LIST/HEAD
fs.glob("s3://growlytics-data/system/inputs/*.csv")
```

This is also why `pandas.read_csv("s3://...")` "just works" — pandas and pyarrow use fsspec under the hood. Install the backend you need (`pip install s3fs`).

**Option 2 — universal_pathlib / cloudpathlib (a `pathlib` mental model).** Since your code is already full of `Path` operations, this is the *least invasive* migration. `UPath` (built on fsspec) and `cloudpathlib` give you objects that quack like `pathlib.Path`:

```python
from upath import UPath

root = UPath("s3://growlytics-data")        # or UPath("/home/zhe/Database")
input_path = root / "system" / "inputs" / "file.csv"
text = input_path.read_text()               # network GET behind the scenes
input_path.exists()
list((root / "system" / "inputs").glob("*.csv"))
```

Swap `Path(...)` → `UPath(...)` and a lot of your existing logic survives. Just stay aware that `.glob`, `.exists`, and reads are now *network calls*, not free.

Pick fsspec if you like the "open a stream" model; pick UPath/cloudpathlib if you want to keep your `pathlib`-shaped code. They're not rivals — UPath sits on top of fsspec.

## Credentials and where config lives in the cloud

Two things surprise people coming from local dev:

**Credentials are ambient, never in the path or config.** Locally the SDK reads `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` or `~/.aws/credentials`. In cloud compute you attach an **IAM role** to the workload (EC2/ECS/Lambda/GKE), and the SDK picks those credentials up automatically. The elegant result: the *same code* uses your keys locally and the role's permissions in prod, with nothing secret in your repo or config.

**Config is injected, not edited in place.** You don't edit files inside a built container image — it's immutable. So the layered pattern from before becomes concrete: your committed JSON keeps only the *relative structure* and sensible defaults, and the environment injects the *root URI* via an env var at deploy time. If you want a richer config object in the cloud, the common trick is: an env var holds a config *URI* (`s3://.../config/path.json`), the app fetches it at startup, and secrets come from a parameter store (AWS SSM / Secrets Manager). Env var → config location → config contents.

## Gotchas worth internalizing now

- **No append; write whole objects.** To grow a dataset, write *new* objects (partitioned outputs) rather than appending to one.
- **"Rename" is copy + delete and not atomic.** For safe output, a common pattern is write to a temp key, then copy into place, then drop a `_SUCCESS` marker.
- **Reads/lists cost latency and can fail.** Use the SDK's built-in retries; don't put a remote read in a tight loop assuming it's free. fsspec's `simplecache`/`filecache` can transparently cache remote files locally — great for dev.
- **Modern S3 is read-after-write strongly consistent**, but don't assume that uniformly across providers/operations.

## The clean architecture for you

Given your modular, editable-install setup, the target shape is small and disciplined:

1. **One Settings object** (pydantic-settings) holds a `root` as a *URI string* (read from an env var) plus the relative subpath structure from committed defaults.
2. **One I/O module** — the only place in the codebase that touches storage — takes a URI and returns readers/writers via fsspec (or hands back `UPath` objects). Everything else calls *this module*, never `open()` or `Path` directly. This is the "swap out the file-read components" you anticipated, done **once**, behind a stable interface.
3. Resolving a resource is just `root_uri` joined with a relative subpath, handed to the I/O module. Local: `file:///...`. Cloud: `s3://...`. Same code path.
4. Credentials come from the environment ambiently; never from config.

The one-line principle that makes it all elegant: **centralize I/O behind a single interface, address everything by URI, and inject location + credentials from the environment.** Do that, and your original "change the config and it just works" goal becomes literally true.

Concrete next step: introduce that single I/O module wrapping fsspec, and migrate one read path (say, your `INPUT_DIR` loader) to take a URI. Test it locally against a `file://` root *and* against a real bucket. Once one path works end to end, the rest is mechanical. Want me to sketch that I/O module and the URI-based Settings object together as a starting scaffold?

# `env-var-driven` Config

Happy to give you the full picture. The reason env-var-driven config is "the norm" isn't fashion — it falls out of how cloud deployment actually works, and once you see the structural reason, all the surrounding best practices line up logically. Let me build it from the principle down to the practice, and tie it back to your Databricks experience at the end.

## The root principle: separate config from code

The canonical source here is the **Twelve-Factor App** methodology (from Heroku, ~2011), whose third factor is simply: *store config in the environment*. The reasoning underneath it is what matters.

"Code" is everything that's identical across every place you run the app. "Config" is everything that *varies* between deployments — which database, which storage bucket, which region, log level, credentials. Twelve-factor's litmus test is sharp: **could you open-source your codebase right now, this second, without leaking a single secret or breaking any environment?** If yes, you've cleanly separated config from code. If your repo contains a committed JSON with `ZheRao`'s OneDrive path, the answer is no — that path is *config* that leaked into the *code artifact*.

So the rule is: config that varies between environments must live neither in the code nor in committed files (committed files are *part of* the code artifact), but in the environment the code runs in.

## The structural reason cloud forces this: immutable artifacts

This is the load-bearing insight, and it's exactly what tripped up your "edit the config after migration" plan.

Modern deployment is built on **immutable artifacts** — a container image, a Lambda zip, a machine image. You build the artifact *once* and promote the *same bytes* through dev → staging → production. This is called "build once, deploy many," and it exists because rebuilding per-environment is how you get the classic "it worked in staging but broke in prod" bug: if the prod artifact is a different build, you never truly tested it.

Now follow the logic: if the *identical* artifact runs in dev, staging, and prod, then by definition every environment-specific value must enter from *outside* the artifact, at runtime. You cannot bake it in, because the bytes are frozen and shared. That "outside, at runtime" injection mechanism is, primarily, environment variables. Your plan to edit a config file inside the deployed system breaks precisely because the artifact is immutable — there's nothing to edit; you replace the whole thing or you inject from outside.

So env-var config isn't a style preference. Immutability plus build-once *mathematically requires* externalized, injected config.

## Why env vars specifically — and the honest modern critique

Env vars won as the injection mechanism because they're:

- **Language- and OS-agnostic.** Every runtime reads them with zero parsing.
- **Changeable per-deploy without rebuilding** the artifact.
- **First-class in every orchestrator** — Docker, Kubernetes, ECS, Lambda, systemd, CI all inject them natively.
- **Granular and hard to accidentally commit** (unlike a tempting config file).

But I'd be doing you a disservice to present dogmatic twelve-factor as the final word, because practice has moved on. Env vars have real weaknesses: they're a flat, untyped, string-only namespace that gets unwieldy past a few dozen values, and — importantly — they *leak*. Env vars are inherited by child processes, visible in process listings, and routinely dumped by crash handlers and error trackers. So the modern, nuanced position is: **env vars for non-secret deploy config and pointers; a real secrets system for secrets; a structured, typed config object for shape and validation.** That split is the next section.

## Config vs secrets — the distinction that separates amateurs from pros

Conflating these is the most common mistake.

**Config** is non-sensitive operational values: bucket names, region, log level, timeouts, feature flags, replica counts. Env vars are perfect for these.

**Secrets** are credentials, API keys, DB passwords, tokens, certs. They need more than env vars can offer: encryption at rest, access auditing, rotation, and least-privilege access. Modern handling, from good to best:

- **Secrets managers** — AWS Secrets Manager, AWS SSM Parameter Store (SecureString), GCP Secret Manager, Azure Key Vault, HashiCorp Vault. The app fetches them at startup, or they're mounted as files. You get rotation and an audit trail.
- **Workload identity (the gold standard)** — instead of the app *holding* a secret, the *workload itself is trusted*. AWS IAM roles, GCP workload identity, Azure managed identities. The runtime hands your process short-lived, auto-rotating credentials through a metadata service, and there's no secret in any env var or file at all. This is the "ambient credentials" idea from our last exchange: the same code uses your laptop's credentials locally and the role's permissions in prod, with nothing sensitive committed anywhere.

A secret in a raw env var is fine for a prototype and weak at scale — no rotation, no audit, and it leaks through the channels above.

## The layered precedence model

In practice you don't choose one source; you stack them, highest priority winning:

1. Command-line flags — per-invocation, ad hoc.
2. Environment variables — per-deploy overrides and pointers.
3. Config files — the *structure* and relationships; a base file plus optional per-environment files.
4. Hardcoded defaults — sane out-of-the-box values that make the app runnable.

Each layer has a job. Defaults make it work on a fresh clone; config files express shape; env vars carry what differs per environment; CLI handles one-off overrides. Your committed JSON's correct role in this world is layers 3–4 — *relative structure and defaults only* — while the bucket/root and any credentials arrive via layers 1–2.

## Where env vars actually come from, per environment

The practical map, since the abstract version only gets you so far:

- **Local dev:** a gitignored `.env` file read by `python-dotenv` or `pydantic-settings`, with a committed `.env.example` documenting every key (no values). This template *is* your config documentation.
- **Docker:** `-e VAR=val`, `--env-file`, or `ENV` in the Dockerfile — but never secrets in the Dockerfile, since they're baked permanently into image layers.
- **docker-compose:** `environment:` and `env_file:` keys.
- **Kubernetes:** `env:` in the pod spec, `ConfigMap` objects for non-secret config, `Secret` objects (mounted as env vars or files), often synced from an external secrets manager.
- **Serverless (Lambda, Cloud Functions):** environment variables in the function config, with secrets pulled from the secret manager.
- **CI/CD:** secrets stored in the CI system (e.g., GitHub Actions secrets) and injected as env vars into the pipeline.

This is also exactly why Databricks "didn't feel different" to you. Doing experimental algorithms in notebooks, you hardcoded paths or used widgets, and the platform hid everything: DBFS mounts abstracted storage, and the cluster's identity silently handled auth. The *central-system* version of that same work uses **secret scopes** (Databricks-managed or backed by Azure Key Vault, accessed via `dbutils.secrets.get(scope, key)`), **job parameters** for parameterization, **Unity Catalog** for governed data locations instead of hardcoded paths, environment-specific job configs, and CI/CD via Databricks Asset Bundles. The config discipline was always there; as an experimenter you simply never had to touch it. That's your gap, and it's a normal one.

## The disciplines that make it elegant

A few cross-cutting practices tie it all together:

**Validate at startup — fail fast.** A misconfigured app should crash immediately and loudly, not twenty minutes into a job when it lazily reads a missing variable. Validate and type-check all config the moment the process boots. This is precisely what `pydantic-settings` gives you: a single typed Settings class that reads from env vars, `.env` files, and secret sources; marks fields required or optional; validates; and applies defaults. It's the concrete tool for layers 1–4 plus fail-fast plus the schema you wanted earlier.

**Define config once, read it everywhere.** One Settings object is the single source of truth. Scattering `os.environ.get(...)` across modules recreates the discoverability problem you'd have with neither approach — nobody can see what knobs exist. Prefix your variables (`GROWLYTICS_`, as you already instinctively did) and use UPPER_SNAKE_CASE to avoid collisions.

**Keep environments at parity; let values differ, not code.** Twelve-factor's tenth factor is dev/prod parity. The *structure* of config should be identical everywhere; only the *values* change. Sprinkling `if env == "prod":` branches through your code is a smell — it means prod runs code paths dev never exercised. Prefer config values that *drive* behavior over environment-name conditionals.

**Distinguish deploy-time from runtime config.** Env vars are set once at deploy and fixed for the process's life. A more advanced tier — feature flags and dynamic config services (LaunchDarkly, AWS AppConfig) — can change *while the app runs*, without redeploy. You almost certainly don't need this yet, but it's the next ring outward, and knowing the distinction keeps you from forcing every changeable value into an env var.

## The one-paragraph synthesis

Env-var-driven config is the cloud norm because cloud deployment runs immutable artifacts promoted unchanged across environments, and immutability *requires* that anything environment-specific be injected from outside at runtime. Env vars are the universal, orchestrator-native injection mechanism for that. Modern practice then refines the dogma: env vars for non-secret deploy config and pointers, a secrets manager plus workload identity for anything sensitive, and a typed config object (pydantic-settings) for structure, defaults, and fail-fast validation — all layered with clear precedence. The unifying discipline is the same idea four ways: separate config from code, define it once, validate it at startup, and inject it from the environment.

If you'd like, I can turn this into a concrete reference scaffold for Growlytics — a single `Settings` class showing each layer (defaults, committed structure file, env overrides, a secret fetched from a manager), wired so it runs identically on your laptop and in a container. That would make every principle above land as code you can actually run.

# Separating Code, Config + Development Cycles

This is the moment everything ties together, so let me give you a real scaffold first and then unpack the four conceptual pieces around it. I'll build the `Settings` class as a file you can drop straight into your package.Locally you can run `python -m growlytics_platform.config` and see everything resolve; set `GROWLYTICS_STORAGE_ROOT=s3://growlytics-prod` first and watch every URI repoint with no code change. That single behavior is the whole lesson in miniature. Now the four pieces around it.

## The mental unlock: code, config, and data are three separate planes

Your recurring confusion — "where do I put external, mutable info if not *imported from within*?" — dissolves once you stop thinking of a running system as one thing. It's three:

The **code plane** is your logic — the transformations, the algorithms. It's frozen into the immutable artifact. It contains *no* environment-specific values.

The **config plane** is the small set of values that tell the code *where* and *how* to run — the storage root, the DB URL, the log level. It lives *outside* the artifact and is handed to the process at startup.

The **data plane** is the actual bytes — your dataframes, tables, files. They live in storage or a database, *also* outside the artifact, reached using addresses from the config plane.

The metaphor that makes it click: your artifact is a sealed appliance, and the environment is the wall socket it's plugged into. The appliance doesn't carry its own electricity — it draws whatever the socket supplies. Plug the same toaster into a 120V or 240V socket and it behaves differently, with zero change to the toaster. "Imported from within" is asking the appliance to contain its own power station. Config isn't imported; **the process is born into an environment that already has the values set on it, and it reads them as it starts.** Your earlier attempt failed because you put a config-plane value (a path) and a data-plane address (where the OneDrive folder is) into the code plane (a committed file). Three planes collapsed into one.

## How config actually gets injected — the concrete channels

So "outside the artifact, handed in at startup" — by whom, through what? These are the real mechanisms, from simplest to most managed:

- **The process environment.** Something sets variables on the shell/process before Python starts (`export GROWLYTICS_STORAGE_ROOT=...`, or a gitignored `.env` your `Settings` loads). The OS holds them; your process inherits them at birth.
- **The orchestrator injects them.** `docker run -e VAR=val`, compose's `env_file:`, a Kubernetes `ConfigMap` (non-secret) or `Secret` mapped into `env:`, a Lambda's environment-variables config. You declare the values in the deployment spec, not the image.
- **Files mounted at runtime.** Kubernetes can mount a `Secret`/`ConfigMap` as files into a known path; the app reads from that mount. The file isn't in the image — it appears in the container at run time. (This is the preferred way to deliver secrets, since mounted files don't leak the way env vars can.)
- **Fetched over the network at startup.** An env var holds a *pointer* (a secret's name, or `s3://.../config.json`), and the app calls a secrets manager or reads that object as it boots. Solves the bootstrap "how do I know where the config is" problem: env var → location → contents.
- **CLI args / job parameters.** Per-invocation values — `argparse`, or in your world a Databricks job parameter / Airflow param. Highest precedence, most ad hoc.

In every case the program never reaches *inside itself* for these values; the surrounding environment supplies them.

## How the system creates and stores dataframes

This is the data plane, and the rule is clean: **config tells you the address; the I/O layer moves the bytes; storage holds them.** A concrete flow:

```python
import pandas as pd
from growlytics_platform.config import get_settings

cfg = get_settings()
df = pd.read_parquet(f"{cfg.inputs_uri}/orders.parquet")   # URI from config
result = transform(df)                                      # logic from code plane
result.to_parquet(f"{cfg.outputs_uri}/orders_clean.parquet")  # bytes to data plane
```

The dataframe *in memory* is ephemeral runtime state — it lives in the process's RAM and vanishes when the job ends. You never "store config-like things"; you *persist data artifacts* to a durable backend, at a location config told you. For an analytics system the default durable backend is **columnar files (Parquet) on object storage**, usually partitioned (e.g. `outputs/date=2026-06-16/...`) so you append by writing *new* objects rather than mutating one. When you outgrow loose files — needing transactions, schema evolution, time-travel — you move up to a **table format like Delta Lake or Iceberg** layered over the same object storage (Delta is exactly what Databricks gave you for free, which is part of why storage felt invisible there). Relational/warehouse backends (Postgres, BigQuery, Snowflake) are the other option when you want SQL and concurrent writers. In all cases the *connection details* are config; the *rows* are data; your *transformations* are code.

## Patterns that keep the planes apart for good

Four habits do almost all the work:

A **single `Settings` object** (the file above) is the only thing that touches `os.environ` or config files. Everything else imports `get_settings()`. This is what stops config from re-blending into the code by a thousand scattered `os.environ.get` calls.

An **I/O / repository layer** is the only thing that touches storage or the database. Business logic asks this layer ("give me the orders dataframe"); it never constructs a path or a connection itself. Swap object storage for a warehouse and only this one module changes.

**Dependency injection** means you build the clients once at startup *from* `Settings` (a storage filesystem, a DB engine) and *pass them in* to the functions and classes that need them, rather than having those reach out to a global. The payoff is testing: in a unit test you inject a fake in-memory store and your logic runs with no cloud and no real DB.

**Backing services as attached resources** (twelve-factor's fourth factor) means the database, object store, and queue are all just URLs-plus-credentials in config. A local Postgres and a cloud RDS instance are *the same code*, different connection string. That interchangeability is the proof your separation is real.

## The system lifecycle: dev → CI → QA/staging → prod

Finally, the thing all of this serves. A mature system moves through distinct environments, and the golden rule running through them is **build once, deploy many**: you build *one* immutable artifact, and only the injected config differs between stages.

- **Local / dev** — your machine, fast iteration, small or fake data, a `.env` file. `GROWLYTICS_ENVIRONMENT=dev`.
- **CI (continuous integration)** — on every push, an automated pipeline runs your tests and *builds the artifact*. Nothing reaches further unless tests pass. This is where the "could I open-source this without leaking secrets" discipline gets enforced.
- **QA / staging** — a production-*like* environment running the **exact same artifact** that will go to prod, but pointed at its own isolated data and config (`GROWLYTICS_ENVIRONMENT=qa`, `GROWLYTICS_STORAGE_ROOT=s3://growlytics-qa`). You validate here because it mirrors prod. The reason staging catches real bugs is precisely that the artifact is identical — only config and data differ.
- **Production** — real data, real consumers (`...=prod`, `s3://growlytics-prod`, secrets from the manager, credentials from a workload role).

Two things to internalize. First, **each environment has its own config values and its own data**, so QA literally cannot corrupt prod's storage — environment separation is data separation. Your `Settings` class is the same in all four; only the injected values change. Second, in a data system "deploy to prod" often means *registering a scheduled job or pipeline* (Airflow, Databricks Workflows) rather than launching a web server — but the lifecycle and the config-injection rules are identical. The promotion path is: version control → CI builds and tests the artifact → deploy that artifact to staging → validate → promote the *same* artifact to prod, with observability (logs, metrics) and a rollback path around it.

That's the complete picture: three planes kept apart, config injected from the environment through concrete channels, data persisted to a backend addressed by that config, and one artifact promoted across a parity-preserving lifecycle. Your `config.py` is the keystone — once the I/O layer reads its URIs and nothing else touches the environment directly, the separation you've been reaching for is structurally guaranteed rather than maintained by discipline.

If you'd like, the natural next artifact is the matching I/O module — the thin fsspec-based layer that turns those URIs into dataframe reads and writes — so you can see the config and data planes connect end to end.

## Appendix: `config.py`

```py
"""
Centralised configuration for the Growlytics platform.

This is the SINGLE place configuration is defined. Nothing else in the codebase
should call os.environ.get(...) or read a config file directly — everything
imports `get_settings()` from here. One source of truth, defined once, read
everywhere.

Config is LAYERED. Highest priority wins:

    1. Environment variables   (GROWLYTICS_*)        <- injected per deployment
    2. A local .env file       (gitignored)          <- developer convenience only
    3. Field defaults below    (committed in code)   <- structure + safe fallbacks

Two deliberate design choices:
  * Locations are stored as URIs (file://, s3://, gs://...) so the SAME code runs
    locally and in the cloud. This module only RESOLVES locations; the separate
    I/O layer turns a URI into an actual reader/writer (fsspec / UPath).
  * Secrets are typed `SecretStr` so they never leak into logs, tracebacks, or repr().
"""

from __future__ import annotations

from functools import lru_cache
from typing import Literal

from pydantic import Field, SecretStr, computed_field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="GROWLYTICS_",     # GROWLYTICS_STORAGE_ROOT -> storage_root
        env_file=".env",              # auto-loaded in local dev; ignored if absent
        env_file_encoding="utf-8",
        extra="ignore",               # don't choke on unrelated env vars
    )

    # ---- Layer 1/2: injected per environment ------------------------------
    # In qa/prod these come from env vars / a secret store. The default here is
    # only a local-dev convenience — ALWAYS inject these in real deployments.
    environment: Literal["dev", "qa", "prod"] = "dev"
    storage_root: str = Field(
        default="file:///tmp/growlytics",
        description="Base URI for all data: 'file:///home/zhe/Database' locally, "
                    "'s3://growlytics-prod' in the cloud.",
    )

    # ---- Layer 3: committed structure + safe defaults ---------------------
    # These are part of the app's SHAPE, rarely change, so they live in code.
    inputs_subpath: str = "system/inputs"
    outputs_subpath: str = "system/outputs"
    consolidated_subpath: str = "system/consolidated"
    user_input_subpath: str = "UserInputs"
    log_level: str = "INFO"

    # ---- Backing services + secrets ---------------------------------------
    # A "backing service" (the DB) is reached via a URL in config, so it's
    # swappable per environment with zero code change. The password is a
    # SecretStr: printing it shows '**********', never the value.
    db_url: str | None = None
    db_password: SecretStr | None = None

    # ---- Validation: fail fast at startup, not mid-job --------------------
    @field_validator("storage_root")
    @classmethod
    def _must_be_uri(cls, v: str) -> str:
        if "://" not in v:
            raise ValueError(
                f"storage_root must be a URI with a scheme "
                f"(file://, s3://, gs://...). Got: {v!r}"
            )
        return v.rstrip("/")

    # ---- Resolved locations (root + relative structure) -------------------
    # Returned as URIs. The I/O layer consumes these; config never does I/O.
    def _join(self, subpath: str) -> str:
        return f"{self.storage_root}/{subpath.lstrip('/')}"

    @computed_field
    @property
    def inputs_uri(self) -> str:
        return self._join(self.inputs_subpath)

    @computed_field
    @property
    def outputs_uri(self) -> str:
        return self._join(self.outputs_subpath)

    @computed_field
    @property
    def consolidated_uri(self) -> str:
        return self._join(self.consolidated_subpath)

    @computed_field
    @property
    def user_input_uri(self) -> str:
        return self._join(self.user_input_subpath)

    @property
    def is_prod(self) -> bool:
        return self.environment == "prod"


@lru_cache
def get_settings() -> Settings:
    """Build and validate settings ONCE; share the same instance everywhere.

    Construction reads env vars + .env and validates types. If a required value
    is missing or malformed, this raises immediately — fail fast at startup
    instead of deep inside a running job.
    """
    return Settings()


if __name__ == "__main__":
    # Smoke test: `python -m growlytics_platform.config` to confirm an
    # environment validates and to see where everything resolves to.
    s = get_settings()
    print(f"environment     : {s.environment}")
    print(f"storage_root    : {s.storage_root}")
    print(f"inputs_uri      : {s.inputs_uri}")
    print(f"outputs_uri     : {s.outputs_uri}")
    print(f"consolidated_uri: {s.consolidated_uri}")
    print(f"db_password     : {s.db_password}")  # -> '**********' or None, never the value
```