---
layout: default
title: Deploy QRcodes on VRSCTEST
nav_order: 3
parent: Runbooks
---

# Deploy QRcodes on VRSCTEST

**Purpose:** Build and deploy the Verus Identity QR Creator on the VRSCTEST testnet, accessible via `https://qrcodes.vrsctest.buildwithdreams.com`.

**Why:** The QR Creator generates Verus identity QR codes (i.e., self-sovereign identity documents). It is a Node.js/TypeScript web application that proxies Verus daemon calls for signing and verification workflows.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**QR Creator:** `10.200.0.13:3000`
**URL:** `https://qrcodes.vrsctest.buildwithdreams.com`

---

## Prerequisites

- VRSCTEST daemon **running** (`08b-start-vrsctest.yml` complete, `docker ps` healthy)
- HITL approval obtained
- `verus-identity-update-qr-creator-main` repo will be cloned on BWD (if not already present)

---

## IP Addressing Convention

All chains use the same final-octet convention within their `/24` Docker network:

| Role | IP |
|---|---|
| Daemon | `.11` |
| RPC server | `.12` |
| QR Creator | `.13` |

VRSC mainnet and VRSCTEST use identical octets on their respective networks — no IP conflict between them.

---

## Port Binding

| Service | Internal port | External host port |
|---|---|---|
| VRSCTEST QR Creator | `3000` | **None** (Docker network only) |

The QR Creator has **no host port binding** — it is reachable only from within `net-vrsctest`. Caddy (which is connected to both `net-vrsc-blue` and `net-vrsctest`) proxies all external HTTPS traffic to it over the Docker network.

This avoids a port conflict with the mainnet QR Creator, which already binds `127.0.0.1:3000` on the same host.

---

## Architecture

{: .warning }
> **Caddy is dual-homed.** After running `37-qrcodes-caddy-network.yml`, the Caddy container (`mains_blue_caddy-caddy-1`) is connected to both `net-vrsc-blue` and `net-vrsctest` simultaneously. This is the mechanism that allows a single Caddy instance to route traffic to upstreams on either network based on the requested domain name.

```
Internet
  │
  ▼
Caddy (mains_blue_caddy-caddy-1)
  ├─ net-vrsc-blue: 10.201.0.10  (existing)
  └─ net-vrsctest:  auto-assigned  (added by playbook 37)
        │
        ▼
  qrcodes.vrsctest.buildwithdreams.com
        │
        ▼
  QR Creator container (dev200_qr-qr-1)
    at 10.200.0.13:3000 (net-vrsctest only)
```

---

## Procedure

### Step 1 — Clone repository

```
Delegate: ansible-playbook -i inventory.ini playbooks/34-qrcodes-clone.yml
```

This playbook:
1. Clones `VerusCoin/verus-identity-update-qr-creator-main` to `~/verus-identity-update-qr-creator-main`
2. Is **idempotent** — skips if already present; does `git pull` to update

### Step 2 — Build Docker image

```
Delegate: ansible-playbook -i inventory.ini playbooks/35-qrcodes-build.yml
         -e qrcodes_rebuild=true   (force clean build, bypass Docker cache)
```

This playbook:
1. Builds `buildwithdreams/verus-qr-creator:vrsctest` (and tags `latest`)
2. Uses multi-stage Docker build: `node:20-alpine` builder → `node:20-alpine` runtime
3. Runs `yarn install --immutable && yarn build` inside the builder stage
4. Skips if image exists and `qrcodes_rebuild` is not set

> Use `-e qrcodes_rebuild=true` when the source has changed or the image is stale.

### Step 3 — Deploy container

```
Delegate: ansible-playbook -i inventory.ini playbooks/36-qrcodes-deploy.yml
```

This playbook:
1. Removes any existing container (`dev200_qr-qr-1`)
2. Writes `docker-compose.yml` targeting `net-vrsctest` at IP `10.200.0.13`
3. Runs `docker compose up -d --force-recreate`
4. Waits for container to appear in `docker ps`

### Step 4 — Connect Caddy to net-vrsctest

```
Delegate: ansible-playbook -i inventory.ini playbooks/37-qrcodes-caddy-network.yml
```

This playbook:
1. Verifies Caddy container (`mains_blue_caddy-caddy-1`) is running
2. Runs `docker network connect net-vrsctest <container>` to attach Caddy to the VRSCTEST network
3. Caddy is **already on `net-vrsc-blue`** — after this it is multi-homed on both networks
4. Is **idempotent** — skips if already connected

> Caddy must be on `net-vrsctest` to be able to reach `10.200.0.13:3000`.

### Step 5 — Add Caddy route

```
Delegate: ansible-playbook -i inventory.ini playbooks/38-qrcodes-caddy-route.yml
```

This playbook:
1. Appends `qrcodes.vrsctest.buildwithdreams.com` route block to the Caddyfile using `blockinfile`
2. Validates the Caddyfile before reloading
3. Runs `docker exec caddy caddy reload` to apply the new route
4. Is **idempotent** — safe to re-run; route block already present on subsequent runs

### Step 6 — Verify

Check the deployed URL:
```bash
curl -I https://qrcodes.vrsctest.buildwithdreams.com
```

Expected: `HTTP/2 200` from the QR Creator's Express server.

To verify container health from the server:
```bash
ssh <host> "docker ps | grep qr"
ssh <host> "curl -s http://10.200.0.13:3000/ | head"
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Caddy not yet connected to `net-vrsctest` | Run `37-qrcodes-caddy-network.yml` |
| `Connection refused` | QR Creator container not running | Run `36-qrcodes-deploy.yml`; check `docker logs dev200_qr-qr-1` |
| `curl: (6) Could not resolve host` | DNS not propagated | Wait 5-10 minutes for Let's Encrypt DNS propagation |
| Container keeps restarting | Health check failing | `docker logs dev200_qr-qr-1`; rebuild with `-e qrcodes_rebuild=true` |
| Route not responding | Caddyfile stale | Run `38-qrcodes-caddy-route.yml` again to reload |

**Container restart loop:**
```bash
docker rm -f dev200_qr-qr-1
# Then re-run Step 3
```

---

## Rollback

To remove the QR Creator container:
```bash
ssh bwd "docker rm -f dev200_qr-qr-1"
```

To remove the Caddy route (disables the domain only, container keeps running):
Edit `~/caddy/Caddyfile` and remove the `qrcodes.vrsctest.buildwithdreams.com` block, then:
```bash
ssh bwd "docker exec mains_blue_caddy-caddy-1 caddy reload --config /config/caddy/Caddyfile"
```

To detach Caddy from `net-vrsctest`:
```bash
ssh bwd "docker network disconnect net-vrsctest mains_blue_caddy-caddy-1"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `34` | `34-qrcodes-clone.yml` | Clone repo |
| `35` | `35-qrcodes-build.yml` | Build Docker image |
| `36` | `36-qrcodes-deploy.yml` | Deploy container on `net-vrsctest` at `10.200.0.13` |
| `37` | `37-qrcodes-caddy-network.yml` | Attach Caddy to `net-vrsctest` |
| `38` | `38-qrcodes-caddy-route.yml` | Add HTTPS route in Caddy |
| `08b` | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |

---

## History

- **2026-04-28** — Created. Follows the same IP-numbering convention as VRSC mainnet (`10.201.0.13` → `10.200.0.13`), with a dedicated Caddy route on `qrcodes.vrsctest.buildwithdreams.com`.
