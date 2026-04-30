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
| 2 | `34b-qrcodes-vrsctest-configure.yml` | Write `config.js` + `Dockerfile` to the clone |
| 3 | `35-qrcodes-vrsctest-build.yml` | Build Docker image |
| 4 | `36-qrcodes-vrsctest-deploy.yml` | Deploy container on `net-vrsctest` |
| 5 | `37-qrcodes-vrsctest-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 6 | `38-qrcodes-vrsctest-caddy-route.yml` | Add HTTPS route in Caddy |

---

## Step 1 — Clone Repository

```bash
cd ~/dream-pbaas-provisioning
ansible-playbook -i inventory.ini playbooks/34-qrcodes-vrsctest-clone.yml
```

Clones `BuildWithDreams/verus-identity-update-qr-creator-main` to `~/verus-identity-update-qr-creator-main` on BWD. Skips if already present.

---

## Step 2 — Configure (`config.js` + `Dockerfile`)

```bash
ansible-playbook -i inventory.ini playbooks/34b-qrcodes-vrsctest-configure.yml
```

This step is required because the repo has no `config.js` and no `Dockerfile` — both are generated from VRSCTEST credentials at deploy time.

**What it writes:**

- `config.js` — contains `RPC_HOST`, `RPC_PORT`, `RPC_USER`, `RPC_PASSWORD` sourced from `vrsctest.conf` on BWD. Baked into the image at build time.
- `Dockerfile` — 2-stage Alpine-based build:
  - **Builder:** `node:20-alpine` + git + yarn; runs `yarn install` (fetches `git+https://` deps from GitHub) then `yarn build`
  - **Runtime:** `node:20-alpine` + curl; copies `dist/`, `node_modules/`, `views/`, `public/`, `config.js` from builder

Re-run this whenever you want to refresh credentials or the `Dockerfile` template.

---

## Step 3 — Build Docker Image

```bash
ansible-playbook -i inventory.ini playbooks/35-qrcodes-vrsctest-build.yml
```

Builds `buildwithdreams/verus-qr-creator:vrsctest`. Skips if image exists and clean build not needed.

**Force rebuild** (after `34b` changes, or source updates):
```bash
ansible-playbook -i inventory.ini playbooks/35-qrcodes-vrsctest-build.yml -e qrcodes_rebuild=true
```

---

## Step 4 — Deploy Container

```bash
ansible-playbook -i inventory.ini playbooks/36-qrcodes-vrsctest-deploy.yml
```

- Removes any existing `dev200_qr-qr-1` container
- Writes `docker-compose.yml` targeting `net-vrsctest` at fixed IP `10.200.0.13`
- Starts container with healthcheck (`wget localhost:3000`), restart `unless-stopped`

**No host port is bound** — the container is accessible only within `net-vrsctest`. Caddy reaches it over the Docker network.

---

## Step 5 — Connect Caddy to `net-vrsctest`

```bash
ansible-playbook -i inventory.ini playbooks/37-qrcodes-vrsctest-caddy-network.yml
```

Attaches the existing Caddy container to `net-vrsctest`. Caddy is already on `net-vrsc-blue` — after this it is multi-homed on both networks. Idempotent.

---

## Step 6 — Add Caddy Route

```bash
ansible-playbook -i inventory.ini playbooks/38-qrcodes-vrsctest-caddy-route.yml
```

1. Pre-flight: verifies Caddy is on `net-vrsctest`; aborts if not
2. Appends `qrcodes.vrsctest.buildwithdreams.com` route block to the Caddyfile (idempotent — skips if already present)
3. Validates Caddyfile syntax
4. Reloads Caddy
5. Health-checks upstream from inside Caddy; warns if QR Creator is unreachable

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
| `502 Bad Gateway` | Caddy not yet on `net-vrsctest` | Run step 5 |
| `Connection refused` | Container not running or unreachable | `docker logs dev200_qr-qr-1`; run step 4 |
| Container restarting, `MODULE_NOT_FOUND` | `node_modules` not in image | Rebuild: run step 3 with `-e qrcodes_rebuild=true` |
| Container restarting, `Cannot find module 'express'` | Same as above | Same fix |
| Container restarting, `Failed to lookup view "index"` | `views/` missing from image | Rebuild: run step 3 with `-e qrcodes_rebuild=true` |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5–10 min for Let's Encrypt DNS |
| Route not responding | Caddyfile stale | Run step 6 to reload |
| `rpcuser` extraction failed | `vrsctest.conf` not readable | Check VRSCTEST daemon is running and conf file exists at `~/.docker-verusd/vrsctest/data_dir/vrsctest.conf` |

**Container in restart loop — clear and rebuild:**
```bash
ssh <bwd-host> "docker rm -f dev200_qr-qr-1"
# Then re-run steps 3–4
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
| 34b | `34b-qrcodes-vrsctest-configure.yml` | Write `config.js` + `Dockerfile` |
| 35 | `35-qrcodes-vrsctest-build.yml` | Build Docker image |
| 36 | `36-qrcodes-vrsctest-deploy.yml` | Deploy container at `10.200.0.13` |
| 37 | `37-qrcodes-vrsctest-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| 38 | `38-qrcodes-vrsctest-caddy-route.yml` | Add HTTPS route |
| 08b | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |

---

## History

- **2026-04-30** — Rewrite. Added step 34b (configure) which generates `config.js` and `Dockerfile` — neither existed in the source repo. Repo URL corrected to `BuildWithDreams/...`. Dockerfile now includes `views/` and `public/` dirs at runtime. Image tagged `vrsctest` only (not `latest`). All playbooks renamed with `-vrsctest-` infix. Playbook 37/38 template escaping fixed (Jinja2 `{{ }}` vs Go template conflict).
- **2026-04-28** — Created.
