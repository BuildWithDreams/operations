---
layout: default
title: Deploy QRcodes on VRSCTEST
nav_order: 3
parent: Runbooks
---

# Deploy QRcodes on VRSCTEST

**Purpose:** Build and deploy the Verus Identity QR Creator on VRSCTEST, accessible at `https://qrcodes.vrsctest.buildwithdreams.com`.

**What it is:** A Node.js/TypeScript web app that generates Verus identity QR codes. It proxies Verus daemon calls for signing and verification workflows. The app reads VRSCTEST RPC credentials from `config.js` (written at deploy time) and connects to the VRSCTEST daemon at `10.200.0.11:18843`.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Upstream:** `10.200.0.13:3000` (Docker network only, no host port)
**URL:** `https://qrcodes.vrsctest.buildwithdreams.com`

---

## Architecture

```
Internet
  │
  ▼
Caddy (mains_blue_caddy-caddy-1)
  ├─ net-vrsc-blue: 10.201.0.10  (existing)
  └─ net-vrsctest: auto-assigned  (added by playbook 37)
        │
        ▼
  qrcodes.vrsctest.buildwithdreams.com
        │
        ▼
  QR Creator container (dev200_qr-qr-1)
    at 10.200.0.13:3000
    image: buildwithdreams/verus-qr-creator:vrsctest
```

Caddy is dual-homed — one container on two networks. It routes `qrcodes.vrsctest.*` to the VRSCTEST upstream based on the Host header, and all other routes to mainnet upstreams.

---

## IP Addressing Convention

| Role | IP (VRSCTEST) | IP (VRSC mainnet) |
|------|---------------|-------------------|
| Daemon | `.11` | `.11` |
| RPC server | `.12` | `.12` |
| QR Creator | `.13` | `.13` |

Same final octets on separate `/24` networks — no conflict between chains.

---

## Prerequisites

- VRSCTEST daemon **running** and **synced** (`docker ps` shows `vrsctest` healthy)
- HITL approval obtained
- Repo `BuildWithDreams/verus-identity-update-qr-creator-main` will be cloned to BWD on first run

---

## Playbook Chain

Run in order. All are idempotent — safe to re-run.

| Step | Playbook | Action |
|------|----------|--------|
| 1 | `34-qrcodes-vrsctest-clone.yml` | Clone repo to BWD |
| 2 | `49-fetch-signer-wif.yml` | Fetch SERVICE_SIGNER_WIF from running daemon |
| 3 | `34b-qrcodes-vrsctest-configure.yml` | Write `config.js` + `Dockerfile` with RPC + signer credentials |
| 4 | `35-qrcodes-vrsctest-build.yml` | Build Docker image |
| 5 | `36-qrcodes-vrsctest-deploy.yml` | Deploy container on `net-vrsctest` |
| 6 | `37-qrcodes-vrsctest-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 7 | `38-qrcodes-vrsctest-caddy-route.yml` | Add HTTPS route in Caddy |

---

## Step 1 — Clone Repository

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/34-qrcodes-vrsctest-clone.yml
```

Clones `BuildWithDreams/verus-identity-update-qr-creator-main` to `~/verus-identity-update-qr-creator-main` on BWD. Skips if already present.

---

## Step 2 — Fetch SERVICE_SIGNER_WIF

```bash
ansible-playbook -i inventory.ini playbooks/49-fetch-signer-wif.yml
```

Fetches the WIF private key for the `BWD@` identity's primary r-address from the running `dev200-vrsctest-1` container.

**What it does:**
1. Runs `verus -chain=VRSCTEST getidentity BWD@` to get the primary r-address
2. Runs `verus -chain=VRSCTEST dumpprivkey <r-address>` to get the WIF
3. Writes the WIF to `~/.bwd_signer_wif` on BWD (mode 0600, never committed)

**Outputs:**
- `SERVICE_SIGNER_WIF` → written to `~/.bwd_signer_wif`
- `SERVICE_SIGNER_IADDRESS` → stored as `bwd_service_signer_iaddress` in `group_vars/production.yml`

The WIF is **never stored in inventory or code**. It must be re-fetched whenever the daemon wallet is encrypted/locked, or on first setup.

**Override identity name** (if needed):
```bash
ansible-playbook -i inventory.ini playbooks/49-fetch-signer-wif.yml -e "identity_name=SomeOther@"
```

---

## Step 3 — Configure (`config.js` + `Dockerfile`)

```bash
ansible-playbook -i inventory.ini playbooks/34b-qrcodes-vrsctest-configure.yml
```

**Requires:** playbook 49 to have been run at least once (WIF file must exist).

**What it writes:**

- `config.js` — contains:
  - `RPC_HOST`, `RPC_PORT`, `RPC_USER`, `RPC_PASSWORD` from `vrsctest.conf`
  - `SERVICE_SIGNER_IADDRESS` from `group_vars/production.yml` (`bwd_service_signer_iaddress`)
  - `SERVICE_SIGNER_WIF` read from `~/.bwd_signer_wif`
- `Dockerfile` — 2-stage Alpine-based build:
  - **Builder:** `node:20-alpine` + git + yarn; runs `yarn install` (fetches `git+https://` deps from GitHub) then `yarn build`
  - **Runtime:** `node:20-alpine` + curl; copies `dist/`, `node_modules/`, `views/`, `public/`, `config.js` from builder

**`config.js` values for BWD@:**

| Key | Value |
|-----|-------|
| `RPC_HOST` | `10.200.0.11` |
| `RPC_PORT` | `18843` |
| `SERVICE_SIGNER_IADDRESS` | `i7o9p3uRfNdtuG8GptWBPhAf4rK39bhJp1` |
| `SERVICE_SIGNER_WIF` | *(fetched at runtime — never stored)* |

Re-run this step whenever: credentials change, the identity rotates, or after `48-qrcodes-vrsctest-update.yml` has been run.

---

## Step 4 — Build Docker Image

```bash
ansible-playbook -i inventory.ini playbooks/35-qrcodes-vrsctest-build.yml
```

Builds `buildwithdreams/verus-qr-creator:vrsctest`. Skips if image exists and clean build not needed.

**Force rebuild** (after `34b` changes, or source updates):
```bash
ansible-playbook -i inventory.ini playbooks/35-qrcodes-vrsctest-build.yml -e qrcodes_rebuild=true
```

---

## Step 5 — Deploy Container

```bash
ansible-playbook -i inventory.ini playbooks/36-qrcodes-vrsctest-deploy.yml
```

- Removes any existing `dev200_qr-qr-1` container
- Writes `docker-compose.yml` targeting `net-vrsctest` at fixed IP `10.200.0.13`
- Starts container with healthcheck (`wget localhost:3000`), restart `unless-stopped`

**No host port is bound** — the container is accessible only within `net-vrsctest`. Caddy reaches it over the Docker network.

---

## Step 6 — Connect Caddy to `net-vrsctest`

```bash
ansible-playbook -i inventory.ini playbooks/37-qrcodes-vrsctest-caddy-network.yml
```

Attaches the existing Caddy container to `net-vrsctest`. Caddy is already on `net-vrsc-blue` — after this it is multi-homed on both networks. Idempotent.

---

## Step 7 — Add Caddy Route

```bash
ansible-playbook -i inventory.ini playbooks/38-qrcodes-vrsctest-caddy-route.yml
```

1. Pre-flight: verifies Caddy is on `net-vrsctest`; aborts if not
2. Appends `qrcodes.vrsctest.buildwithdreams.com` route block to the Caddyfile (idempotent — skips if already present)
3. Validates Caddyfile syntax
4. Reloads Caddy
5. Health-checks upstream from inside Caddy; warns if QR Creator is unreachable

---

## Update (Pull + Rebuild)

To update an existing deployment after source changes:

```bash
ansible-playbook -i inventory.ini playbooks/48-qrcodes-vrsctest-update.yml
```

This convenience wrapper chains:
1. Pull latest from GitHub
2. Re-fetch signer WIF from daemon
3. Re-write `config.js` with fresh WIF
4. Rebuild Docker image (`--no-cache`)
5. Force-recreate container

**Requires:** playbook 34 (clone) must have been run at least once.

---

## Verify

```bash
curl -I https://qrcodes.vrsctest.buildwithdreams.com
```

Expected: `HTTP/2 200`, `content-type: text/html`

```bash
ssh <bwd-host> "docker ps --filter 'name=dev200_qr'"
ssh <bwd-host> "docker logs dev200_qr-qr-1 --tail 5"
```

Expected: `Up` with `(healthy)` status.

---

## Full Re-deploy (all steps)

After any source change or when starting fresh:

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/34-qrcodes-vrsctest-clone.yml
ansible-playbook -i inventory.ini playbooks/49-fetch-signer-wif.yml
ansible-playbook -i inventory.ini playbooks/34b-qrcodes-vrsctest-configure.yml
ansible-playbook -i inventory.ini playbooks/35-qrcodes-vrsctest-build.yml -e qrcodes_rebuild=true
ansible-playbook -i inventory.ini playbooks/36-qrcodes-vrsctest-deploy.yml
ansible-playbook -i inventory.ini playbooks/37-qrcodes-vrsctest-caddy-network.yml
ansible-playbook -i inventory.ini playbooks/38-qrcodes-vrsctest-caddy-route.yml
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `502 Bad Gateway` | Caddy not yet on `net-vrsctest` | Run step 6 |
| `Connection refused` | Container not running or unreachable | `docker logs dev200_qr-qr-1`; run step 5 |
| Container restarting, `MODULE_NOT_FOUND` | `node_modules` not in image | Rebuild: run step 4 with `-e qrcodes_rebuild=true` |
| Container restarting, `Cannot find module 'express'` | Same as above | Same fix |
| Container restarting, `Failed to lookup view "index"` | `views/` missing from image | Rebuild: run step 4 with `-e qrcodes_rebuild=true` |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5–10 min for Let's Encrypt DNS |
| Route not responding | Caddyfile stale | Run step 7 to reload |
| `rpcuser` extraction failed | `vrsctest.conf` not readable | Check VRSCTEST daemon is running and conf file exists |
| `SERVICE_SIGNER_WIF` fetch fails | Daemon wallet is locked | Unlock wallet: `verus -chain=VRSCTEST walletpassphrase <pass> 60` then re-run step 2 |
| Identity not found on fetch | Wrong identity name | Run `verus -chain=VRSCTEST getidentity <identity@>` manually to verify |

**Container in restart loop — clear and rebuild:**
```bash
ssh <bwd-host> "docker rm -f dev200_qr-qr-1"
# Then re-run steps 4–5
```

---

## Rollback

**Remove container only:**
```bash
ssh <bwd-host> "docker rm -f dev200_qr-qr-1"
```

**Remove Caddy route (container keeps running):**
Edit `~/caddy/Caddyfile` on BWD, remove the `qrcodes.vrsctest.buildwithdreams.com` block, then:
```bash
ssh <bwd-host> "docker exec mains_blue_caddy-caddy-1 caddy reload --config /config/caddy/Caddyfile"
```

**Detach Caddy from `net-vrsctest`:**
```bash
ssh <bwd-host> "docker network disconnect net-vrsctest mains_blue_caddy-caddy-1"
```

**Remove image:**
```bash
ssh <bwd-host> "docker rmi buildwithdreams/verus-qr-creator:vrsctest"
```

---

## Relevant Playbooks

| # | File | Purpose |
|---|------|---------|
| 34 | `34-qrcodes-vrsctest-clone.yml` | Clone repo to BWD |
| 34b | `34b-qrcodes-vrsctest-configure.yml` | Write `config.js` + `Dockerfile` with RPC + signer credentials |
| 35 | `35-qrcodes-vrsctest-build.yml` | Build Docker image |
| 36 | `36-qrcodes-vrsctest-deploy.yml` | Deploy container at `10.200.0.13` |
| 37 | `37-qrcodes-vrsctest-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 38 | `38-qrcodes-vrsctest-caddy-route.yml` | Add HTTPS route |
| 48 | `48-qrcodes-vrsctest-update.yml` | Update (pull + rebuild + restart) |
| 49 | `49-fetch-signer-wif.yml` | Fetch SERVICE_SIGNER_WIF from daemon (reusable) |
| 08b | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |

---

## History

- **2026-05-04** — Add service signer feature. New playbook 49 (fetch WIF) and 48 (update wrapper). Updated 34b to write `SERVICE_SIGNER_IADDRESS` and `SERVICE_SIGNER_WIF` to `config.js`. IADDRESS stored as `bwd_service_signer_iaddress` in `group_vars/production.yml`. WIF never stored — always fetched from running daemon at runtime.
- **2026-04-30** — Rewrite. Added step 34b (configure) which generates `config.js` and `Dockerfile` — neither existed in the source repo. Repo URL corrected to `BuildWithDreams/...`. Dockerfile now includes `views/` and `public/` dirs at runtime. Image tagged `vrsctest` only (not `latest`). All playbooks renamed with `-vrsctest-` infix. Playbook 37/38 template escaping fixed (Jinja2 `{{ }}` vs Go template conflict).
- **2026-04-28** — Created.
