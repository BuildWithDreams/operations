---
layout: default
title: Deploy svc-idcreate on VRSC mainnet
nav_order: 4
parent: Deployment
---

# Deploy svc-idcreate on VRSC mainnet

**Purpose:** Build and deploy the Verus Identity Creation service on VRSC mainnet, accessible via `https://idcreate.vrsc.buildwithdreams.com`.

**Why:** The idcreate service exposes the Verus identity registration API — create commitments, submit signed requests, and track provisioning status. It connects directly to the VRSC mainnet daemon via RPC to sign and submit on-chain identity transactions.

**Chain:** VRSC mainnet
**Network:** `net-vrsc-blue` (`10.201.0.0/24`) — Caddy; `net-vrsctest` (`10.200.0.0/24`) — idcreate containers
**Service IP:** `10.200.0.14` (idcreate containers on `net-vrsctest`; Caddy reaches them via its `net-vrsctest` interface at `10.200.0.2`)
**URL:** `https://idcreate.vrsc.buildwithdreams.com`

---

## Prerequisites

- VRSC mainnet daemon **running** (`08-start-vrsc.yml` complete, `docker ps` healthy on `net-vrsc-blue`)
- `rpcallowip=10.201.0.0/24` already set in VRSC `.conf` (playbook `16` or previous run)
- Caddy already on `net-vrsc-blue` (playbook `28-caddy-deploy.yml` already adds Caddy to `net-vrsc-blue` natively — no join step needed)
- HITL approval obtained
- `BuildWithDreams/svc-idcreate` will be cloned on BWD at `~/svc-idcreate-vrsc` (if not already present)
- DNS A record for `idcreate.vrsc.buildwithdreams.com` must point to `135.181.136.105`

---

## IP Addressing Convention

All chains use the same final-octet convention within their `/24` Docker network:

| Role | IP |
|---|---|
| Daemon | `.11` |
| RPC server | `.12` |
| QR Creator | `.13` |
| idcreate | `.14` |

VRSC mainnet and VRSCTEST use identical octets on their respective networks — no IP conflict between them.

---

## Architecture (v1)

```
Internet
  │
  ▼
Caddy (mains_blue_caddy-caddy-1)
  ├─ net-vrsc-blue:  10.201.0.10
  └─ net-vrsctest:   10.200.0.2    (Caddy is multi-homed — reaches idcreate via this interface)
        │
        ▼
  idcreate.vrsc.buildwithdreams.com
        │
        ▼
  idcreate-api container (dev200_idcreate-api-1)
    at 10.200.0.14:5003 (net-vrsctest network)
        │
        ├──────────────────────────────┐
        ▼                              ▼
  idcreate-worker              idcreate-provisioning
  (background state machine)   (Node.js HTTP adapter)
        │                              │
        ▼                              ▼
  PostgreSQL 16-alpine               svc-provisioning scripts
  dev200_idcreate-postgres-1         (provisioning/engine.py)
  5432 (host: 127.0.0.1:5432)
        │
        ▼
  VRSC mainnet verusd daemon at 10.201.0.11:27486
```

> **Note:** idcreate containers run on `net-vrsctest` (`10.200.0.14`) not `net-vrsc-blue`. Caddy is multi-homed and reaches them via its `net-vrsctest` interface (`10.200.0.2`). No network join step needed.

---

## Containers (v1)

| Container | Service | IP | Port |
|---|---|---|---|
| `dev200_idcreate-postgres-1` | PostgreSQL 16-alpine | auto | `5432` (host: `127.0.0.1:5432`) |
| `dev200_idcreate-provisioning-1` | Node.js provisioning scripts | auto | `5055` (host: `127.0.0.1:5055`) |
| `dev200_idcreate-api-1` | FastAPI | `10.200.0.14` | `5003` (host: `127.0.0.1:5003`) |
| `dev200_idcreate-worker-1` | Polling worker | auto | — (internal) |

> Container names are currently `dev200_idcreate-*` (not `dev201_idcreate-*`) due to a compose project naming issue. tracked in [GitHub issue #24](https://github.com/BuildWithDreams/dream-pbaas-provisioning/issues/24).

---

## Environment Variables Set by Playbook 41b

Playbook `41b-idcreate-deploy-vrsc.yml` reads VRSC RPC credentials from `~/docker-verusd/mainnet/data_dir/VRSC.conf` and writes these to `~/svc-idcreate-vrsc/.env`:

```
NATIVE_COIN=VRSC
HEALTH_RPC_DAEMON=verusd_vrsc
verusd_vrsc_rpc_enabled=true
verusd_vrsc_rpc_user=<from VRSC.conf>
verusd_vrsc_rpc_password=<from VRSC.conf>
verusd_vrsc_rpc_port=27486
verusd_vrsc_rpc_host=10.201.0.11
```

**Postgres vars (v1):**

```
DATABASE_URL=postgresql://idcreate:***@postgres:5432/idcreate
PROVISIONING_ADAPTER_MODE=http
PROVISIONING_SERVICE_URL=http://127.0.0.1:5055
PROVISIONING_HTTP_TIMEOUT_SECONDS=10
PROVISIONING_RETRY_COUNT=1
PROVISIONING_LOG_LEVEL=INFO
```

---

## Procedure

### Step 1 — Clone repository

```
Delegate: ansible-playbook -i inventory.ini playbooks/39b-idcreate-clone-vrsc.yml
```

This playbook:
1. Clones `BuildWithDreams/svc-idcreate` to `~/svc-idcreate-vrsc`
2. Is **idempotent** — skips if already present; does `git pull` to update

### Step 2 — Build Docker image

```
Delegate: ansible-playbook -i inventory.ini playbooks/40b-idcreate-build-vrsc.yml
         -e idcreate_rebuild=true   (force clean build, bypass Docker cache)
```

This playbook:
1. Builds `buildwithdreams/svc-idcreate:local` (and tags `latest`)
2. Uses `docker build --no-cache` from the `~/svc-idcreate-vrsc` Dockerfile
3. Skips if image exists and `idcreate_rebuild` is not set

> Use `-e idcreate_rebuild=true` when the source has changed or the image is stale.

### Step 3 — Deploy containers

```
Delegate: ansible-playbook -i inventory.ini playbooks/41b-idcreate-deploy-vrsc.yml
```

This playbook:
1. Reads VRSC RPC credentials from `~/docker-verusd/mainnet/data_dir/VRSC.conf`
2. Writes `~/svc-idcreate-vrsc/.env` with all required vars (RPC, NATIVE_COIN, DATABASE_URL, provisioning adapter, etc.)
3. Auto-generates a strong PostgreSQL password and writes it to `DATABASE_URL`
4. Removes any existing containers
5. Writes `docker-compose.yml` with 4 services: postgres + provisioning + api + worker
6. Runs `docker compose up -d --force-recreate`
7. Waits for postgres container to become healthy (`pg_isready`)
8. Waits for api container to appear in `docker ps`

> If `.env` already exists, step 1 (copy from env.sample) is skipped and all vars are updated in place — safe to re-run.

### Step 3.5 — Issue API key (if not already set)

```
Delegate: ansible-playbook -i inventory.ini playbooks/43b-idcreate-add-api-key-vrsc.yml
```

This playbook:
1. Checks if `REGISTRAR_API_KEYS` is already in `~/svc-idcreate-vrsc/.env`
2. If missing: generates a `BWD_<32-hex>` key and writes it (idempotent — skips if already present)
3. Prints the key to the ansible output — **copy it now**, it will not be shown again

> After running this, you **must restart the containers** for the new env var to take effect:
> ```
> Delegate: ansible-playbook -i inventory.ini playbooks/47b-idcreate-restart-vrsc.yml
> ```

### Step 4 — Add Caddy route

```
Delegate: ansible-playbook -i inventory.ini playbooks/42b-idcreate-caddy-route-vrsc.yml
```

This playbook:
1. **Pre-flight check** — verifies Caddy is connected to `net-vrsc-blue`; aborts if missing
2. Appends `idcreate.vrsc.buildwithdreams.com` route block to the host Caddyfile at `/home/dream-hermes-agent/caddy/Caddyfile` using `blockinfile`
3. Validates the Caddyfile before reloading (`docker exec caddy caddy validate`)
4. Runs `docker exec caddy caddy reload --config /config/caddy/Caddyfile` to apply the new route
5. **Health check** — hits `http://10.200.0.14:5003/` from inside the Caddy container; warns if unreachable
6. Is **idempotent** — safe to re-run; skips all changes if route already present

> **Caddyfile update convention:** The Caddyfile is mounted read-only inside the container at `/config/caddy/Caddyfile`. Playbooks edit the **host** file at `/home/dream-hermes-agent/caddy/Caddyfile` directly (via `blockinfile`), then validate+reload inside the container. `docker cp` is NOT used — the container sees the host mount without intervention.
>
> **Unlike VRSCTEST:** Caddy is natively on `net-vrsc-blue` (no network join step needed). The pre-flight check still runs but only to confirm, not to join.

### Step 5 — Configure (as needed)

Set the source of funds address (required for the service to operate on-chain):

```
Delegate: ansible-playbook -i inventory.ini playbooks/44b-idcreate-source-of-funds-vrsc.yml -e "source_of_funds_address=RXXXXXXXXXXXXXXXXXXXXXXXX"
```

Set allowed parent currencies:

```
Delegate: ansible-playbook -i inventory.ini playbooks/46b-idcreate-allowed-parents-vrsc.yml -e "currency_ids=VRSC,RVSR"
```

### Step 6 — Verify

Check the deployed URL:
```bash
curl -I https://idcreate.vrsc.buildwithdreams.com
```

Expected: `HTTP/2 200` from the FastAPI service.

To verify container health from the server:
```bash
ssh bwd "docker ps | grep dev200_idcreate"
ssh bwd "curl -s http://10.200.0.14:5003/"
```

To verify PostgreSQL is running:
```bash
ssh bwd "docker ps | grep postgres"
ssh bwd "docker exec dev200_idcreate-postgres-1 pg_isready -U idcreate -d idcreate"
```

To check logs:
```bash
ssh bwd "docker logs dev200_idcreate-api-1 --tail 50"
ssh bwd "docker logs dev200_idcreate-worker-1 --tail 50"
ssh bwd "docker logs dev200_idcreate-postgres-1 --tail 20"
```

---

## Updating idcreate VRSC

To pull latest code and rebuild:

```
Delegate: ansible-playbook -i inventory.ini playbooks/45b-idcreate-update-vrsc.yml
```

This playbook:
1. Backs up `.env` before any operation
2. Fetches `origin/main` and resets to latest (`git fetch + reset --hard`)
3. Restores `.env` from backup after pull (git merge may remove it)
4. Rebuilds the Docker image (always — no cache bypass)
5. Force-recreates all containers
6. Waits for postgres health and api to be running

---

## PostgreSQL Backup

Compose stores DB data on volume `idcreate_postgres_data`.

Recommended: periodic host-level volume snapshots, keep at least daily backups, keep longer retention for operational audits.

Example backup command:
```bash
ssh bwd "cd ~/svc-idcreate-vrsc && docker compose -p dev201_idcreate exec -T postgres pg_dump -U idcreate -d idcreate > idcreate_vrsc_backup.sql"
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Caddy not on `net-vrsc-blue` | Verify `28-caddy-deploy.yml` was run; check `docker network inspect net-vrsc-blue` |
| `Connection refused` | idcreate container not running | Run `41b-idcreate-deploy-vrsc.yml`; check `docker logs dev200_idcreate-api-1` |
| `500 Internal Server Error` | RPC connection failed | Verify VRSC daemon running; check `docker logs dev200_idcreate-api-1` for RPC errors |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5-10 minutes for Let's Encrypt DNS propagation |
| Container keeps restarting | Health check failing or bad env vars | `docker logs dev200_idcreate-api-1`; verify `.env` has correct RPC port and credentials |
| PostgreSQL container not healthy | postgres not ready yet | Wait up to 150s; check `docker logs dev200_idcreate-postgres-1` |
| `idcreate.vrsc.buildwithdreams.com` resolves but returns 404 | Caddy route not added | Run `42b-idcreate-caddy-route-vrsc.yml` |

**Container restart loop:**
```bash
ssh bwd "cd ~/svc-idcreate-vrsc && docker compose -p dev201_idcreate down"
# Then re-run Step 3
Delegate: ansible-playbook -i inventory.ini playbooks/41b-idcreate-deploy-vrsc.yml
```

**RPC connection refused — common cause:**
The VRSC mainnet daemon port is `27486`. Verify:
```bash
ssh bwd "docker exec dev200-vrsc-1 verus-cli getinfo | grep port"
```

---

## Rollback

To remove the idcreate VRSC containers:
```bash
ssh bwd "cd ~/svc-idcreate-vrsc && docker compose -p dev201_idcreate down"
```

To remove the Caddy route (disables the domain only, containers keep running):
```bash
ssh bwd "docker exec mains_blue_caddy-caddy-1 caddy reload --config /config/caddy/Caddyfile"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `08` | `08-start-vrsc.yml` | Start VRSC mainnet daemon (prereq) |
| `16` | `16-add-vrsc-rpc-allowip.yml` | Ensure `rpcallowip=10.201.0.0/24` (prereq) |
| `28` | `28-caddy-deploy.yml` | Deploy Caddy on `net-vrsc-blue` (already done) |
| `39b` | `39b-idcreate-clone-vrsc.yml` | Clone repo to `~/svc-idcreate-vrsc` |
| `40b` | `40b-idcreate-build-vrsc.yml` | Build Docker image |
| `41b` | `41b-idcreate-deploy-vrsc.yml` | Deploy full stack (postgres + provisioning + api + worker) |
| `42b` | `42b-idcreate-caddy-route-vrsc.yml` | Add HTTPS route in Caddy |
| `43b` | `43b-idcreate-add-api-key-vrsc.yml` | Generate and persist `REGISTRAR_API_KEYS` test key |
| `44b` | `44b-idcreate-source-of-funds-vrsc.yml` | Set `SOURCE_OF_FUNDS` (requires `-e`) |
| `45b` | `45b-idcreate-update-vrsc.yml` | Pull + rebuild + restart |
| `46b` | `46b-idcreate-allowed-parents-vrsc.yml` | Set `REGISTRAR_ALLOWED_PARENTS` (requires `-e`) |
| `47b` | `47b-idcreate-restart-vrsc.yml` | Force-recreate containers to pick up `.env` changes |

---

## Conventions

### Caddyfile management

The Caddy container's `/config/caddy/` mount is **read-only** inside the container. Playbooks must:

1. Edit the **host** Caddyfile at `/home/dream-hermes-agent/caddy/Caddyfile` (via `blockinfile` or `lineinfile`)
2. Validate with `docker exec <container> caddy validate --config /config/caddy/Caddyfile`
3. Reload with `docker exec <container> caddy reload --config /config/caddy/Caddyfile`

**Do NOT use `docker cp`** to copy the Caddyfile into the container — it will fail with `read-only file system` errors.

**Do NOT run `caddy fmt` inside the container** — the `/config` path is read-only. If formatting is needed, run `caddy fmt --overwrite` on the host directly.

### Route idempotency checks

Use `grep -Fq '<exact-domain>'` (fixed-string, no substring matches) when checking for existing routes. `grep -q` without `-F` produces false positives when a domain like `idcreate.vrsctest` is present and you search for `idcreate.vrsc`.

### Variable scoping

VRSC-specific values must use prefixed variable names (`idcreate_vrsc_*`) distinct from the shared VRSCTEST defaults (`idcreate_*`). Do not reuse the unprefixed vars for VRSC-specific values — they are used by VRSCTEST playbooks and must remain unchanged.

### Docker network routing

PBaaS chains are isolated by network. A service accessible at `10.x.0.14` on `net-vrsctest` is NOT reachable from `net-vrsc-blue` directly — even if the IP octet is the same. Caddy must be multi-homed (on both networks) to proxy between them. All current Caddy containers are multi-homed; verify before assuming connectivity.

### Compose project naming

Container names are derived from the compose project name plus service name. The project name is set via `COMPOSE_PROJECT_NAME` in the `.env` file, NOT via hardcoded `container_name` in the compose YAML. Hardcoded `container_name` breaks the ability to run multiple chain deployments with different prefixes. See [GitHub issue #24](https://github.com/BuildWithDreams/dream-pbaas-provisioning/issues/24).

---

## History

- **2026-05-16** — Created. Ports the VRSCTEST idcreate deployment pattern to VRSC mainnet. Uses `net-vrsc-blue` for Caddy and `net-vrsctest` for idcreate containers (Caddy is multi-homed). Caddyfile managed via host edits + validate/reload (no docker cp, no fmt inside container). VRSC uses `idcreate_vrsc_*` prefixed vars. Container naming tracked in issue #24.