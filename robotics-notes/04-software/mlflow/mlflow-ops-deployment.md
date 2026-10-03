---
topic: Running a self-hosted MLflow stack — Docker compose, reboot persistence, backup & migration
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Running a self-hosted MLflow stack: ops, persistence, backup, migration

**One-line idea:**
A self-hosted MLflow deployment is a multi-container stack (Postgres + MLflow server + reverse proxy) wired together with Docker Compose; reboot persistence, data durability, and migrating to a new host all come down to *where each piece of state physically lives*.

**Why it exists:**
Once MLflow moves from "local files, one person" to "shared server, whole team," someone has to own keeping the server up, its data safe across restarts/failures, and know how to move it if the host ever changes.

**Math / mechanics:**
Key mechanics:

- **Reboot persistence chain:** containers get a `restart: unless-stopped` policy rather than relying on manual restarts. The Docker daemon itself is a `systemd`-managed service. On host reboot: `systemd` starts → starts the Docker daemon → the daemon reads each container's restart policy → restarts anything marked `unless-stopped`. That policy specifically means "come back after a reboot or crash, but stay down if a human explicitly ran `docker compose down`."
- **State lives in two places with very different durability properties:**
  - *Metadata* (experiments, runs, params, metrics, tags) → a relational database (Postgres), kept in a Docker **named volume on the server's local disk** — deliberately *not* on a network filesystem (NFS), because databases need fast, exclusive file locking that NFS doesn't reliably provide.
  - *Artifacts* (model checkpoints, plots) → written to a shared network filesystem (NFS) or object store, since these are large files rather than queryable data, and shared storage means every machine that mounts it sees them immediately.
  - Consequence: NFS-backed artifacts survive a server disk failure or a full server migration automatically (just remount the same share elsewhere). The Postgres volume does **not** — it's tied to that one host's local disk.
- **Backup = closing the one durability gap.** Because metadata is local-disk-only, the backup strategy is a scheduled `pg_dump` of the database, written out *to the shared NFS* (so the backup itself survives even if the host's local disk fails) — e.g. a nightly cron job.
- **Migrating the whole stack to a new host**, mechanically: (1) `pg_dump` the old Postgres to NFS, (2) bring up the same Compose stack on the new host (same compose files, fresh/empty Postgres volume), (3) restore the dump into the new Postgres via `psql`, (4) point every client's `MLFLOW_TRACKING_URI` (and browser bookmarks) at the new host, (5) regenerate any self-signed TLS cert if it was bound to the old host's IP/hostname. Nothing needs to happen to the artifacts — they're already visible via the same NFS mount on the new host.

**Code:**
```yaml
# docker-compose.yml excerpt
services:
  postgres:
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data   # local named volume, NOT on NFS
  mlflow:
    restart: unless-stopped
  proxy:
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
# backup: dump Postgres metadata to shared NFS (e.g. nightly cron)
0 2 * * * docker exec mycontainer-postgres-1 pg_dump -U mlflow mlflow \
  > /mnt/shared/mlflow_backup_$(date +\%Y\%m\%d).sql

# migration: dump on old host -> restore on new host
docker exec postgres-1 pg_dump -U mlflow mlflow > /mnt/shared/migration.sql   # old host
docker compose up -d                                                          # new host
cat /mnt/shared/migration.sql | docker exec -i postgres-1 psql -U mlflow mlflow   # new host

# verify Docker is set to auto-start after reboot
systemctl is-enabled docker
```

**Gotchas:**
- A multi-container stack recreated partway through a failed `up` can leave a container attached to **no network at all** (not even the Compose-created default network). Symptoms look like a DNS resolution failure between services (e.g. a service can't resolve a sibling by its Compose service name) even though both containers show as "running." Fix is a clean `down`/`up`, not patching the half-started container.
- Bind-mounting a host directory that doesn't exist yet onto an NFS share with `root_squash` fails: Docker tries to create-and-`chown` the missing directory as root on first use, but NFS maps root to an unprivileged user and refuses the `chown`. Fix: pre-create the directory yourself (as your own, non-root user) before bringing the stack up.
- A leftover container from unrelated prior work (e.g. an older container that happens to publish the same host port) can silently prevent a new service from binding that port. Diagnosing "why won't this start" requires checking what else already holds the port (`docker ps`, `ss`/`netstat`), not just the new service's own config.
- A credential baked into a database's initialization (e.g. a Postgres password set via env var) only takes effect on that database's **very first startup against an empty volume**. Changing the env var afterward does nothing unless you wipe and recreate the volume.
- Never run `docker compose down -v` out of habit. The `-v` flag destroys named volumes — i.e. the Postgres data — along with the containers. Plain `down` stops containers but preserves volumes.

**Related:** [[mlflow-tracking-architecture]], [[reverse-proxy-networking-fundamentals]]

**Still unclear:**
These are operational specifics observed on one real deployment (a single Linux host running Docker, with an NFS-mounted shared storage volume). Exact `systemd` unit names and the precise mechanics of how Docker registers itself as a boot-time service weren't independently inspected beyond confirming `systemctl is-enabled docker` reports enabled.
