---
layout: default
title: Deploy svc-idcreate on VRSCTEST
nav_order: 3
parent: Deployment
---

# Deploy svc-idcreate on VRSCTEST

**Purpose:** Build and deploy the Verus Identity Creation service on the VRSCTEST testnet, accessible via `https://idcreate.vrsctest.buildwithdreams.com`.

**Why:** The idcreate service exposes the Verus identity registration API — create commitments, submit signed requests, and track provisioning status. It connects directly to the VRSCTEST daemon via RPC to sign and submit on-chain identity transactions.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Service IP:** `10.200.0.14`
**URL:** `https://idcreate.vrsctest.buildwithdreams.com`

---

## Prerequisites

- VRSCTEST daemon **running** (`08b-start-vrsctest.yml` complete, `docker ps` healthy)
- `rpcallowip=10.200.0.0/24` already set in VRSCTEST `.conf` (playbook `16b` or previous run)
- Caddy already on `net-vrsctest` (added by `37-qrcodes-caddy-network.yml` during qrcodes deployment)
- HITL approval obtained
- `BuildWithDreams/svc-idcreate` will be cloned on BWD (if not already present)

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

## Architecture

```
Internet
  │
  ▼
Caddy (mains_blue_caddy-caddy-1)
  ├─ net-vrsc-blue: 10.201.0.10  (existing)
  └─ net-vrsctest:  auto-assigned  (added by playbook 37, qrcodes deploy)
        │
        ▼
  idcreate.vrsctest.buildwithdreams.com
        │
        ▼
  idcreate-api container (dev200_idcreate-api-1)
    at 10.200.0.14:5003 (net-vrsctest only)
        │
        ▼ (background)
  idcreate-worker container (dev200_idcreate-worker-1)
    polls pending registrations, advances state
        │
        ▼
  VRSCTEST verusd daemon at 10.200.0.11:18842
```

> **Caddy is dual-homed.** It is already connected to `net-vrsctest` from the qrcodes deployment (playbook `37`). No network changes needed for this deployment.

---

## Environment Variables Written by Playbook 41

Playbook `41-idcreate-deploy.yml` reads VRSCTEST RPC credentials from `~/docker-verusd/vrsctest/data_dir/vrsctest.conf` and writes these to `~/svc-idcreate/.env`:

```
NATIVE_COIN=VRSCTEST
HEALTH_RPC_DAEMON=verusd_vrsc
verusd_vrsc_rpc_enabled=true
verusd_vrsc_rpc_user=<from vrsctest.conf>
verusd_vrsc_rpc_password=<from vrsctest.conf>
verusd_vrsc_rpc_port=18842
verusd_vrsc_rpc_host=10.200.0.11
```

> **SFConstants mapping:** The VRSCTEST chain hijacks the `DAEMON_VERUSD_VRSC` slot in `SFConstants.py` via `NATIVE_COIN="VRSCTEST"`. There is no separate VRSCTEST daemon slot.

---

## Containers

| Container | Service | IP | Port |
|---|---|---|---|
| `dev200_idcreate-api-1` | FastAPI | `10.200.0.14` | `5003` (host: `127.0.0.1:5003`) |
| `dev200_idcreate-worker-1` | Polling worker | `10.200.0.14` | — (internal) |

No provisioning service — not ready yet. Focus is on the id creation base Python service.

---

## Procedure

### Step 1 — Clone repository

```
Delegate: ansible-playbook -i inventory.ini playbooks/39-idcreate-clone.yml
```

This playbook:
1. Clones `BuildWithDreams/svc-idcreate` to `~/svc-idcreate`
2. Is **idempotent** — skips if already present; does `git pull` to update

### Step 2 — Build Docker image

```
Delegate: ansible-playbook -i inventory.ini playbooks/40-idcreate-build.yml
         -e idcreate_rebuild=true   (force clean build, bypass Docker cache)
```

This playbook:
1. Builds `buildwithdreams/svc-idcreate:local` (and tags `latest`)
2. Uses `docker build --no-cache` from the `~/svc-idcreate` Dockerfile
3. Skips if image exists and `idcreate_rebuild` is not set

> Use `-e idcreate_rebuild=true` when the source has changed or the image is stale.

### Step 3 — Deploy containers

```
Delegate: ansible-playbook -i inventory.ini playbooks/41-idcreate-deploy.yml
```

This playbook:
1. Reads VRSCTEST RPC credentials from `~/docker-verusd/vrsctest/data_dir/vrsctest.conf`
2. Writes `~/svc-idcreate/.env` with all required vars (RPC, NATIVE_COIN, etc.)
3. Removes any existing containers (`dev200_idcreate-api-1`, `dev200_idcreate-worker-1`)
4. Writes `docker-compose.yml` targeting `net-vrsctest` at IP `10.200.0.14`
5. Runs `docker compose up -d --force-recreate`
6. Waits for api container to appear in `docker ps`

> If `.env` already exists, step 1 (copy from env.sample) is skipped and RPC vars are updated in place — safe to re-run.

### Step 4 — Add Caddy route

```
Delegate: ansible-playbook -i inventory.ini playbooks/42-idcreate-caddy-route.yml
```

This playbook:
1. **Pre-flight check** — verifies Caddy is connected to `net-vrsctest`; aborts if missing
2. Appends `idcreate.vrsctest.buildwithdreams.com` route block to the Caddyfile using `blockinfile`
3. Runs `caddy fmt --overwrite` to format the file cleanly
4. Validates the Caddyfile before reloading (aborts if invalid)
5. Runs `docker exec caddy caddy reload` to apply the new route
6. **Health check** — hits `http://10.200.0.14:5003/health` from inside the Caddy container; warns if unreachable
7. Is **idempotent** — safe to re-run; skips all changes if route already present

> Caddy is already on `net-vrsctest` (added by playbook `37` during qrcodes deployment). Step 4 only adds the route.

### Step 5 — Verify

Check the deployed URL:
```bash
curl -I https://idcreate.vrsctest.buildwithdreams.com
```

Expected: `HTTP/2 200` from the FastAPI service.

To verify container health from the server:
```bash
ssh bwd "docker ps | grep idcreate"
ssh bwd "curl -s http://10.200.0.14:5003/health"
```

To check logs:
```bash
ssh bwd "docker logs dev200_idcreate-api-1 --tail 50"
ssh bwd "docker logs dev200_idcreate-worker-1 --tail 50"
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Caddy not on `net-vrsctest` | Run `37-qrcodes-caddy-network.yml` (should already be done) |
| `Connection refused` | idcreate container not running | Run `41-idcreate-deploy.yml`; check `docker logs dev200_idcreate-api-1` |
| `500 Internal Server Error` | RPC connection failed | Verify VRSCTEST daemon running; check `docker logs dev200_idcreate-api-1` for RPC errors |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5-10 minutes for Let's Encrypt DNS propagation |
| Container keeps restarting | Health check failing or bad env vars | `docker logs dev200_idcreate-api-1`; verify `.env` has correct RPC port and credentials |
| Route not responding | Caddyfile stale | Run `42-idcreate-caddy-route.yml` again to reload |

**Container restart loop:**
```bash
ssh bwd "docker rm -f dev200_idcreate-api-1 dev200_idcreate-worker-1"
# Then re-run Step 3
```

**RPC connection refused — common cause:**
The VRSCTEST daemon port is `18842`, not the standard `27486`. Verify:
```bash
ssh bwd "docker exec dev200-vrsctest-1 verus-cli -chain=VRSCTEST getinfo | grep port"
```

---

## Rollback

To remove the idcreate containers:
```bash
ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate down"
```

To remove the Caddy route (disables the domain only, containers keep running):
Edit `~/caddy/Caddyfile` and remove the `idcreate.vrsctest.buildwithdreams.com` block, then reload:
```bash
ssh bwd "docker exec mains_blue_caddy-caddy-1 caddy reload --config /config/caddy/Caddyfile"
```

To remove the Caddy route via playbook (idempotent — safe):
```bash
Delegate: ansible-playbook -i inventory.ini playbooks/42-idcreate-caddy-route.yml
# Re-running removes nothing (route is already present), but you can manually
# remove the ANSIBLE MANAGED BLOCK from the Caddyfile and reload manually
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `39` | `39-idcreate-clone.yml` | Clone repo |
| `40` | `40-idcreate-build.yml` | Build Docker image |
| `41` | `41-idcreate-deploy.yml` | Deploy api+worker on `net-vrsctest` at `10.200.0.14`; writes `.env` |
| `42` | `42-idcreate-caddy-route.yml` | Add HTTPS route in Caddy |
| `08b` | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |
| `16b` | `16b-add-vrsctest-rpc-allowip.yml` | Ensure `rpcallowip=10.200.0.0/24` (prereq) |
| `37` | `37-qrcodes-caddy-network.yml` | Connect Caddy to `net-vrsctest` (already done) |

---

## History

- **2026-04-29** — Created. Mirrors the qrcodes VRSCTEST deployment pattern. Deploys api + worker only; provisioning service excluded (not ready). VRSCTEST uses the `DAEMON_VERUSD_VRSC` slot via `NATIVE_COIN="VRSCTEST"` — no separate VRSCTEST slot in `SFConstants.py`.
