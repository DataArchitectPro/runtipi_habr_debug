# Runtipi v4.6.5 (Proxmox LXC): verifying connection to an external PostgreSQL

> 🇷🇺 [Читать README на русском](README.ru.md)

## Context
- Proxmox → LXC container → Docker/Compose inside
- Runtipi: `v4.6.5`, directory `/opt/runtipi`
- Runtipi containers: `runtipi`, `runtipi-reverse-proxy`, `runtipi-queue`, `runtipi-db (postgres:14)`
- Goal: switch Runtipi to an external PostgreSQL (another LXC in the same network) and stop using `runtipi-db`.

External PostgreSQL: `192.168.100.106:5432`, database `tipi`, user `tipi`.  
Password (hash) in files: `de99…377b` *(not a secret, but masked in the text just in case)*.

---

## What is included in the archive

The archive contains two “snapshots” of the system:

### `00_Initial_state/` — before any changes
**`00_Initial_state/configs/`**
- `.env` — original Runtipi env file. Key point: `POSTGRES_HOST=runtipi-db`
- `docker-compose.yml` — original compose file (with the `runtipi-db` service)

**`00_Initial_state/docker/`**
- `inspect_runtipi.json` — output of `docker inspect runtipi`
- `inspect_runtipi-db.json` — output of `docker inspect runtipi-db`
- `docker-compose.config.yml` — output of `docker compose config` (effective configuration as seen by Compose)

**`00_Initial_state/logs/`**
- `runtipi.log` — `docker logs runtipi`
- `runtipi-db.log` — `docker logs runtipi-db`

---

### `01_Attempt_external_PostgreSQL/` — after attempting the switch
**`01_Attempt_external_PostgreSQL/configs/`**
- `.env.local` — attempt to configure an external PostgreSQL:
  - `POSTGRES_HOST=192.168.100.106`
  - `POSTGRES_PORT=5432`
  - `POSTGRES_DBNAME=tipi`
  - `POSTGRES_USERNAME=tipi`
  - `POSTGRES_PASSWORD=de99…377b`
- `.env` — **still contains `POSTGRES_HOST=runtipi-db`**
- `docker-compose.yml` — still includes the `runtipi-db` service (`postgres:14`)

**`01_Attempt_external_PostgreSQL/docker/`**
- `inspect_runtipi.json` — `docker inspect runtipi` after changes
- `docker-compose.config.yml` — `docker compose config` after changes (**crucial for conclusions**)

**`01_Attempt_external_PostgreSQL/logs/`**
- `runtipi.log` — `docker logs runtipi`
- `runtipi-db.log` — `docker logs runtipi-db`

**`01_Attempt_external_PostgreSQL/command_outputs/`**
- `env_inside_container.txt` — output of `env` inside the `runtipi` container
- `runtipi-cli_status.txt` — attempt to run `./runtipi-cli status` (“command not found” in this run)

**`01_Attempt_external_PostgreSQL/additional_checks_after_changes.txt`**
- output of `docker ps` and other verification commands after the changes

---

## What the files show (key observations)

### 1) Inside the `runtipi` container, variables point to the external PostgreSQL
File: `01_.../command_outputs/env_inside_container.txt`

It contains:
- `POSTGRES_HOST=192.168.100.106`
- `POSTGRES_PORT=5432`
- `POSTGRES_DBNAME=tipi`
- `POSTGRES_USERNAME=tipi`
- `POSTGRES_PASSWORD=...`

This is also confirmed by:  
File: `01_.../docker/inspect_runtipi.json` → `Config.Env` contains `POSTGRES_HOST=192.168.100.106`.

➡️ So the external DB parameters do reach the container environment.

---

### 2) But Docker Compose “sees” something else: `POSTGRES_HOST: runtipi-db` and `depends_on: runtipi-db`
File: `01_.../docker/docker-compose.config.yml`

It clearly shows:
- `depends_on: runtipi-db`
- `POSTGRES_HOST: runtipi-db`

➡️ This means that **`docker compose config` is not built from `.env.local`, but from the regular `.env` (or a generated environment)**, and still assumes PostgreSQL is a local service.

---

### 3) The `runtipi-db` service is still actually running
File: `01_.../additional_checks_after_changes.txt`

`docker ps` shows:
- `runtipi-db postgres:14 ... Up ... (healthy)`

Additionally, in `01_.../configs/docker-compose.yml`, the `runtipi-db:` service is still present.

➡️ Even after attempts to “move the DB outside”, the local PostgreSQL container remains part of the stack and is started.

---

### 4) Key evidence: the `runtipi` container **bind-mounts** `/opt/runtipi/.env` as `/data/.env`
File: `01_.../docker/inspect_runtipi.json` → `Mounts`

There is a bind mount:
- `Source: /opt/runtipi/.env`
- `Destination: /data/.env`

And the file `/opt/runtipi/.env` (snapshot `01_.../configs/.env`) contains:
- `POSTGRES_HOST=runtipi-db`

In `runtipi` logs there is a line:
- `Generating system env file`

➡️ This strongly suggests that Runtipi, on startup, **generates/overwrites a “system env” based on `/data/.env`**, i.e. based on the actual `/opt/runtipi/.env`, not on `.env.local`.

---

## Final conclusion based on this archive

From the collected data, two facts hold at the same time:
1) the `runtipi` container does receive `POSTGRES_*` variables pointing to the external host;
2) but the Runtipi / Compose stack remains tied to the local `runtipi-db`, because:
   - the effective Compose configuration (`docker compose config`) still contains `POSTGRES_HOST=runtipi-db` and `depends_on: runtipi-db`;
   - `runtipi` mounts `/opt/runtipi/.env` as `/data/.env`, and that file still points to `runtipi-db`.

In other words, switching via `.env.local` looks **insufficient / workaround-like**. One of the following is likely required:
- modifying `/opt/runtipi/.env` directly (the file that is actually used),
- changing the startup / env-generation logic inside Runtipi,
- or fully removing the `runtipi-db` service from the compose stack and checking whether the CLI / generator recreates it.

---

## Questions for Runtipi experts
1) What is the “canonical” way to configure an external PostgreSQL so that Runtipi **does not regenerate env back to `runtipi-db`**?
2) Where is the single source of truth for DB configuration: container env vars, `/data/.env`, some `system.env`, state file, or user config?
3) If `--env-file` is used, at which stage is it applied (CLI vs Docker Compose), and why does `docker compose config` still resolve to `runtipi-db`?

---

## Files with the most relevant evidence
- `01_.../docker/inspect_runtipi.json` — mounts + env inside the container
- `01_.../docker/docker-compose.config.yml` — what Compose actually resolves (`POSTGRES_HOST=runtipi-db`)
- `01_.../command_outputs/env_inside_container.txt` — env inside the container (external IP visible)
- `01_.../additional_checks_after_changes.txt` — `docker ps`, showing that `runtipi-db` is still running
- `01_.../configs/.env` and `.env.local` — conflicting sources of truth
