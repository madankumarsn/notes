---
topic: MLflow tracking server architecture (client-server, backend store, artifact store)
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# MLflow tracking server architecture

**One-line idea:**
MLflow is a client-server system — training scripts and browsers are both just HTTP clients hitting one `mlflow server` process, which splits storage into a metadata **backend store** (a database) and a file **artifact store**, organized as Tracking backend → Experiment → Run.

**Why it exists:**
Teams running many training experiments need a shared, queryable record of what hyperparameters produced which results and where the resulting model artifacts live, instead of everyone managing their own flat per-run folders.

**Math / mechanics:**

- **Hierarchy:** Tracking backend → Experiment (named bucket of related runs) → Run (one execution, unique auto-generated `run_id`) → `{params, metrics (time series keyed by step), tags, artifacts}`.
- **Two separate stores, different jobs:**
  - *Backend store* — a database holding params/metrics/tags/run metadata. SQLite is fine for one local user; once multiple jobs/viewers write concurrently from different machines, you need Postgres (or MySQL) — SQLite doesn't handle concurrent multi-writer access safely and risks "database is locked" errors or corruption.
  - *Artifact store* — actual files (model checkpoints, plots). Too large for a database row, so it's written to a directory or object store instead.
- **One server process serves both the API and the UI.** `mlflow server` exposes a REST API (`/api/2.0/mlflow/...`) for programmatic read/write *and* a React-based web UI from the same process; the browser downloads that UI's HTML/JS, which then calls the same REST API. There's no separate "frontend server."
- **`MLFLOW_TRACKING_URI`** (or `--mlflow-tracking-uri`) is the single thing that decides where a client's `mlflow.log_metric()`/`log_artifact()` calls go: unset → local SQLite/file fallback; set to an `http(s)://` URL → every call becomes an HTTP POST to that server instead. No other code change is needed to switch from solo/local to shared/team mode.
- **`--serve-artifacts`** makes the server proxy artifact uploads/downloads itself, so clients never need direct filesystem/object-store credentials. This is what avoids the "absolute path baked into the database" breakage you get from just copying a raw SQLite file + local `mlruns/` folder between machines.
- **MLflow 3.5+ added two security middlewares** on the server that silently reject otherwise-valid requests unless explicitly configured:
  - `--allowed-hosts` validates the HTTP `Host` header (anti DNS-rebinding). Defaults to `localhost`/private IPs only — a client connecting via a Docker alias like `host.docker.internal:5000` gets a `403` until that host is allow-listed.
  - `--cors-allowed-origins` validates the browser `Origin` header on API calls. Defaults to localhost only — a browser hitting the UI via any other hostname/IP gets `403` on every **POST**, while GETs (loading the page shell) still succeed. This produces a deceptive partial failure: the UI loads, but no data renders.

**Code:**
```bash
# Start a server with a real (shared) backend + proxied artifacts
mlflow server \
  --backend-store-uri postgresql://user:pw@host:5432/mlflow \
  --artifacts-destination file:/mlartifacts \
  --serve-artifacts \
  --allowed-hosts "*" \
  --cors-allowed-origins "*" \
  --host 0.0.0.0 --port 5000

# Client side — no code change needed beyond setting this:
export MLFLOW_TRACKING_URI=http://pika-server1:5000
```

```bash
# Explicit, deterministic local fallback (don't rely on cwd-relative defaults)
sqlite:///$(pwd)/mlflow.db
mlflow ui --backend-store-uri sqlite:///$(pwd)/mlflow.db
```

**Gotchas:**
- The local-fallback default has changed across MLflow versions: older docs/assumptions point at a `./mlruns` file store, but that store is now deprecated and raises if used explicitly; the current default is a cwd-relative `sqlite:///mlflow.db`. Relying on "wherever the process happens to run" is fragile — pin an explicit absolute tracking URI instead.
- **Run names are not unique identifiers**, only a display tag. Re-running the same named config produces multiple runs with the same name but different `run_id`s and timestamps — sort/filter by start time or tags if you need "the" run, or append a timestamp to the name if you want uniqueness.
- A single flat experiment with many runs works until you want different families kept apart — switch experiments via a different experiment-name per family (e.g. an env var), not via subfolders; MLflow has no sub-grouping within an experiment.
- `host.docker.internal:PORT` as a `Host` header is rejected by MLflow 3.5+'s default allowed-hosts list even though it resolves to a private IP — the check matches the literal header string, not the resolved address.
- "Page loads but no data" is specifically a CORS `403` on POST endpoints — a different failure mode than a Host-header `403`, which blocks the page itself from loading at all. Telling these apart (check browser devtools network tab / server logs for which requests are `403`) is the key diagnostic step.

**Related:** [[reverse-proxy-networking-fundamentals]], [[mlflow-ops-deployment]]

**Still unclear:**
This is drawn from one real, detailed debugging session on one project's self-hosted MLflow setup (MLflow roughly 3.5–3.13 era based on what came up). Version-specific defaults (e.g. the exact MLflow release that introduced the security middleware) weren't independently verified against MLflow's own changelog — only what surfaced while debugging this one deployment.
