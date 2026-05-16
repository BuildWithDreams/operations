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
**Network:** `net-vrsc-blue` (`10.201.0.0/24`)
**Service IP:** `10.201.0.14`
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
  └─ net-vrsc-blue: 10.201.0.10  (already native — no network join needed)
        │
        ▼
  idcreate.vrsc.buildwithdreams.com
        │
        ▼
  idcreate-api container (dev201_idcreate-api-1)
    at 10.201.0.14:5003 (net-vrsc-blue only)
        │
        ├──────────────────────────────┐
        ▼                              ▼
  idcreate-worker              idcreate-provisioning
  (background state machine)   (Node.js HTTP adapter)
        │                              │
        ▼                              ▼
  PostgreSQL 16-alpine               svc-provisioning scripts
  dev201_idcreate-postgres-1         (provisioning/engine.py)
  5432 (host: 127.0.0.1:5432)
        │
        ▼
  VRSC mainnet verusd daemon at 10.201.0.11:27486
```

---

## Containers (v1)

| Container | Service | IP | Port |
|---|---|---|---|
| `dev201_idcreate-postgres-1` | PostgreSQL 16-alpine | auto | `5432` (host: `127.0.0.1:5432`) |
| `dev201_idcreate-provisioning-1` | Node.js provisioning scripts | auto | `5055` (host: `127.0.0.1:5055`) |
| `dev201_idcreate-api-1` | FastAPI | `10.201.0.14` | `5003` (host: `127.0.0.1:5003`) |
| `dev201_idcreate-worker-1` | Polling worker | auto | — (internal) |

---

## Environment Variables Set by Playbook 41b

Playbook `41b-idcreate-deploy-vrsc.yml` reads VRSC RPC credentials from `~/docker-verusd/vrsc/data_dir/VRSC.conf` and writes these to `~/svc-idcreate-vrsc/.env`:

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
1. Reads VRSC RPC credentials from `~/docker-verusd/vrsc/data_dir/VRSC.conf`
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
2. Appends `idcreate.vrsc.buildwithdreams.com` route block to the host Caddyfile using `blockinfile`
3. Copies the updated Caddyfile into the container (`docker cp` — needed because the container mount is read-only)
4. Validates the Caddyfile before reloading (aborts if invalid)
5. Runs `docker exec caddy caddy reload` to apply the new route
6. **Health check** — hits `http://10.201.0.14:5003/health` from inside the Caddy container; warns if unreachable
7. Is **idempotent** — safe to re-run; skips all changes if route already present

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
ssh bwd "docker ps | grep dev201_idcreate"
ssh bwd "curl -s http://10.201.0.14:5003/health"
```

To verify PostgreSQL is running:
```bash
ssh bwd "docker ps | grep postgres"
ssh bwd "docker exec dev201_idcreate-postgres-1 pg_isready -U idcreate -d idcreate"
```

To check logs:
```bash
ssh bwd "docker logs dev201_idcreate-api-1 --tail 50"
ssh bwd "docker logs dev201_idcreate-worker-1 --tail 50"
ssh bwd "docker logs dev201_idcreate-postgres-1 --tail 20"
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
| `Connection refused` | idcreate container not running | Run `41b-idcreate-deploy-vrsc.yml`; check `docker logs dev201_idcreate-api-1` |
| `500 Internal Server Error` | RPC connection failed | Verify VRSC daemon running; check `docker logs dev201_idcreate-api-1` for RPC errors |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5-10 minutes for Let's Encrypt DNS propagation |
| Container keeps restarting | Health check failing or bad env vars | `docker logs dev201_idcreate-api-1`; verify `.env` has correct RPC port and credentials |
| PostgreSQL container not healthy | postgres not ready yet | Wait up to 150s; check `docker logs dev201_idcreate-postgres-1` |
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
ssh bwd "docker exec dev201-vrsc-1 verus-cli getinfo | grep port"
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

## History

- **2026-05-16** — Created. Ports the VRSCTEST idcreate deployment pattern to VRSC mainnet. Uses `net-vrsc-blue` instead of `net-vrsctest`. Caddy is natively on `net-vrsc-blue` (no join step needed). VRSC uses `idcreate_vrsc_*` vars in `group_vars/production-local.yml`. All VRSC-specific values are isolated from VRSCTEST values.