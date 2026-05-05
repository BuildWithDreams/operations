---
layout: default
title: Deploy s-nomp on VRSCTEST
nav_order: 4
parent: Deployment
---

# Deploy s-nomp on VRSCTEST

**Purpose:** Build and deploy the VerusCoin s-nomp mining pool on VRSCTEST testnet, accessible at `https://pool.vrsctest.buildwithdreams.com` for the web UI, and `stratum+tcp://<server>:3334` for miners.

**What it is:** s-nomp (simple Node Open Mining Portal) is the VerusCoin community's mining pool software. It connects to the VRSCTEST daemon via RPC, runs a stratum server for miners to submit shares, and serves a web dashboard showing pool stats, hashrate, and earnings.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Pool container:** `10.200.0.15:3000` (web UI, internal only)
**Stratum ports:** `3333` (localhost) / `3334` (internet)
**URL:** `https://pool.vrsctest.buildwithdreams.com`

---

## Architecture

```
Internet
  │
  ├── Miners → stratum+tcp://<server>:3334 (workers connect directly)
  │
  ▼
Caddy (mains_blue_caddy-caddy-1)
  ├─ net-vrsc-blue: 10.201.0.10
  └─ net-vrsctest: auto-assigned       ← attached by playbook 53
        │
        ▼
  pool.vrsctest.buildwithdreams.com
        │
        ▼
  s-nomp container (dev200_snomp-snomp-1)
    at 10.200.0.15:3000
    image: buildwithdreams/s-nomp:vrsctest
    daemon: 10.200.0.11:18843 (VRSCTEST daemon)
```

---

## IP Addressing Convention

All chains use the same final-octet convention within their `/24` Docker network:

| Role | IP (VRSCTEST) |
|------|---------------|
| Daemon | `.11` |
| RPC server | `.12` |
| QR Creator | `.13` |
| idcreate | `.14` |
| **s-nomp** | **`.15`** |

---

## Prerequisites

- VRSCTEST daemon **running** and **synced** (`docker ps` shows `dev200-vrsctest-1` healthy)
- `rpcallowip=10.200.0.0/24` set in VRSCTEST `.conf` (playbook `16b` or previous run)
- Caddy already deployed (`28-caddy-deploy.yml`)
- HITL approval obtained
- Pool R-address for receiving fees and payouts (operator provides)
- `VerusCoin/s-nomp` will be cloned on first run

---

## Playbook Chain

Run in order. All are idempotent — safe to re-run.

| Step | Playbook | Action |
|------|----------|--------|
| 1 | `50-s-nomp-clone.yml` | Clone `VerusCoin/s-nomp` to BWD |
| 2 | `50b-s-nomp-configure.yml` | Write `config.json` + `website.json` + `Dockerfile` |
| 3 | `51-s-nomp-build.yml` | Build Docker image |
| 4 | `52-s-nomp-deploy.yml` | Deploy container, expose stratum ports |
| 5 | `53-s-nomp-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 6 | `54-s-nomp-caddy-route.yml` | Add HTTPS route in Caddy |

---

## Step 1 — Clone Repository

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/50-s-nomp-clone.yml
```

Clones `VerusCoin/s-nomp` (master branch) to `~/s-nomp` on BWD. Skips if already present — does a `git fetch + reset` to keep it up to date.

---

## Step 2 — Configure

```bash
ansible-playbook -i inventory.ini playbooks/50b-s-nomp-configure.yml \
  -e "pool_address=RTEST..."
```

**Requires:** VRSCTEST daemon must be running so `vrsctest.conf` exists with RPC credentials.

**What it writes:**

- `config.json` — pool configuration targeting:
  - Daemon RPC: `http://10.200.0.11:18843` (reads credentials from `vrsctest.conf`)
  - Payment processing: payout to `pool_address`
  - Pool fee: `0.5%`
  - Stratum ports: `3333` (internal), `3334` (external)
- `website.json` — web UI branding (title, colors, fee display)
- `Dockerfile` — Alpine-based multi-stage build: `node:20-alpine` builder → `node:20-alpine` runtime

**Parameters:**

| Parameter | Description | Required |
|-----------|-------------|----------|
| `pool_address` | R-address for pool fees and payouts | Yes |
| `pool_fee` | Pool fee % (default: `0.5`) | No |
| `pool_passphrase` | Pool passphrase (default: empty) | No |

---

## Step 3 — Build Docker Image

```bash
ansible-playbook -i inventory.ini playbooks/51-s-nomp-build.yml
```

Builds `buildwithdreams/s-nomp:vrsctest`. Skips if image already exists.

**Force rebuild** (after Step 2 changes or source updates):
```bash
ansible-playbook -i inventory.ini playbooks/51-s-nomp-build.yml -e s_nomp_rebuild=true
```

---

## Step 4 — Deploy Container

```bash
ansible-playbook -i inventory.ini playbooks/52-s-nomp-deploy.yml \
  -e "pool_address=RTEST..."
```

- Removes any existing container
- Starts `dev200_snomp-snomp-1` on `net-vrsctest` at `10.200.0.15`
- Exposes stratum ports:
  - `3333` → `127.0.0.1:3333` (localhost only — for local test miners)
  - `3334` → `0.0.0.0:3334` (internet — for remote workers)
- Web UI at port `3000` is **not** exposed on the host — only accessible via Caddy over Docker network

> **Stratum port security:** Port `3334` binds to `0.0.0.0` intentionally so external miners can connect. The VRSCTEST network is a testnet — no real funds at risk. For mainnet, restrict this to specific IP ranges via firewall rules.

---

## Step 5 — Connect Caddy to `net-vrsctest`

```bash
ansible-playbook -i inventory.ini playbooks/53-s-nomp-caddy-network.yml
```

Attaches the existing Caddy container to `net-vrsctest`. Caddy is already on `net-vrsc-blue` — after this it is multi-homed on both networks. Idempotent.

---

## Step 6 — Add Caddy Route

```bash
ansible-playbook -i inventory.ini playbooks/54-s-nomp-caddy-route.yml
```

1. Pre-flight: verifies Caddy is on `net-vrsctest`; aborts if not
2. Appends `pool.vrsctest.buildwithdreams.com` route block to the Caddyfile (idempotent — skips if already present)
3. Validates Caddyfile syntax
4. Reloads Caddy
5. Health-checks upstream; warns if s-nomp is unreachable

---

## Verify

**Web UI:**
```bash
curl -I https://pool.vrsctest.buildwithdreams.com
```
Expected: `HTTP/2 200`, `content-type: text/html`

**Container:**
```bash
ssh <bwd-host> "docker ps --filter 'name=dev200_snomp'"
ssh <bwd-host> "docker logs dev200_snomp-snomp-1 --tail 10"
```

**Stratum port:**
```bash
ssh <bwd-host> "ss -tlnp | grep -E '3333|3334'"
```
Expected: port `3334` listening on `0.0.0.0`

**Pool stats endpoint (internal):**
```bash
curl -s http://10.200.0.15:3000/api/poolStats
```

---

## Miner Connection

Give workers this connection string:

```
stratum+tcp://pool.vrsctest.buildwithdreams.com:3334
```

Or for miners behind firewalls that block port `3334`, use port `3333` via localhost tunnel (local testing only):

```
stratum+tcp://localhost:3333
```

**Worker config example (NBMiner, for Verus/Equihash):**
```json
{
  "algo": "equihash",
  "pool1": "stratum+tcp://pool.vrsctest.buildwithdreams.com:3334",
  "wallet": "RTEST...your-address",
  "email": "worker@email"
}
```

---

## Update (Pull + Rebuild)

To update after source changes:

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/50-s-nomp-clone.yml
ansible-playbook -i inventory.ini playbooks/50b-s-nomp-configure.yml -e "pool_address=RTEST..."
ansible-playbook -i inventory.ini playbooks/51-s-nomp-build.yml -e s_nomp_rebuild=true
ansible-playbook -i inventory.ini playbooks/52-s-nomp-deploy.yml -e "pool_address=RTEST..."
```

---

## Full Re-deploy (all steps)

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/50-s-nomp-clone.yml
ansible-playbook -i inventory.ini playbooks/50b-s-nomp-configure.yml -e "pool_address=RTEST..."
ansible-playbook -i inventory.ini playbooks/51-s-nomp-build.yml -e s_nomp_rebuild=true
ansible-playbook -i inventory.ini playbooks/52-s-nomp-deploy.yml -e "pool_address=RTEST..."
ansible-playbook -i inventory.ini playbooks/53-s-nomp-caddy-network.yml
ansible-playbook -i inventory.ini playbooks/54-s-nomp-caddy-route.yml
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `502 Bad Gateway` | Caddy not on `net-vrsctest` | Run step 5 |
| `Connection refused` on stratum | Container not running | `docker logs dev200_snomp-snomp-1`; re-run step 4 |
| `stratum+tcp` connection refused | Port not open on host | `ss -tlnp \| grep 3334`; check firewall |
| Pool shows `0 hashrate` | Daemon not syncing or RPC broken | Check `10.200.0.11:18843` RPC; restart daemon |
| `curl: (6) Could not resolve host` | Let's Encrypt DNS not propagated | Wait 5–10 min |
| Route not responding | Caddyfile stale | Run step 6 to reload |
| `rpcuser` extraction failed | `vrsctest.conf` not readable | Check VRSCTEST daemon is running |
| Container in restart loop | `config.json` error | `docker logs dev200_snomp-snomp-1`; re-run step 2 + 3 |

**Container in restart loop — clear and rebuild:**
```bash
ssh <bwd-host> "docker rm -f dev200_snomp-snomp-1"
# Then re-run steps 3–4
```

---

## Rollback

**Remove container only:**
```bash
ssh <bwd-host> "docker rm -f dev200_snomp-snomp-1"
```

**Remove Caddy route (container keeps running):**
Edit `~/caddy/Caddyfile` on BWD, remove the `pool.vrsctest.buildwithdreams.com` block, then:
```bash
ssh <bwd-host> "docker exec mains_blue_caddy-caddy-1 caddy reload --config /config/caddy/Caddyfile"
```

**Detach Caddy from `net-vrsctest`:**
```bash
ssh <bwd-host> "docker network disconnect net-vrsctest mains_blue_caddy-caddy-1"
```

**Remove image:**
```bash
ssh <bwd-host> "docker rmi buildwithdreams/s-nomp:vrsctest"
```

---

## Relevant Playbooks

| # | File | Purpose |
|---|------|---------|
| 50 | `50-s-nomp-clone.yml` | Clone `VerusCoin/s-nomp` to BWD |
| 50b | `50b-s-nomp-configure.yml` | Write `config.json` + `website.json` + `Dockerfile` |
| 51 | `51-s-nomp-build.yml` | Build Docker image |
| 52 | `52-s-nomp-deploy.yml` | Deploy container at `10.200.0.15`, expose stratum ports |
| 53 | `53-s-nomp-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 54 | `54-s-nomp-caddy-route.yml` | Add HTTPS route for pool web UI |
| 08b | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |
| 16b | `16b-add-vrsctest-rpc-allowip.yml` | Allow RPC from `10.200.0.0/24` (prereq) |

---

## History

- **2026-05-05** — Created. Initial s-nomp deployment for VRSCTEST testnet. Pool at `pool.vrsctest.buildwithdreams.com`, miners connect via `stratum+tcp://<server>:3334`.
