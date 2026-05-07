---
layout: default
title: Check sync status
nav_order: 1
parent: Runbooks
---

# Check sync status

**Purpose:** Verify that VRSC and vDEX chains on BWD are fully synced.

**Why:** Determines if chains are healthy before deployment, trading, or other operations that require chain integrity.

**Chains:** VRSC (mainnet) + vDEX (PBaaS)

---

## Prerequisites

- BWD server is reachable
- Containers are running (playbook will report offline if not)

---

## Procedure

### Step 1 — Run sync check

```
Delegate: ansible-playbook -i inventory.ini playbooks/15-sync-status.yml
```

### Step 2 — Interpret output

Look for the status table:

```
║ Chain         ║ Local / Tip   ║ Gap      ║ Peers / TLS    ║
║ VRSC          ║ 4056739 / 4056739 ║      +0 ║   5 / 5       ║
║ vDEX          ║ 0897765 / 0897765 ║      +0 ║   5 / 5       ║
```

**Healthy:** `Local == Tip`, `Gap: +0`, `Peers > 0`

**Out of sync:** `Local < Tip`, gap is positive — chain is behind

**Offline:** Shows `offline` instead of block numbers

---

## What the columns mean

| Column | What it means |
|--------|---------------|
| `Local / Tip` | Local block height vs network longest chain |
| `Gap` | `+N` = N blocks behind; `0` = fully synced |
| `Peers` | Number of peer connections |
| `TLS` | Encrypted peer connections (should match Peers for full security) |

---

## If a chain is offline

Check container status:

```
Delegate: ansible -i inventory.ini -m shell -a "docker ps -a | grep -E 'vrsc|vdex'" 135.181.136.105
```

Then start the chain:

```
Delegate: ansible-playbook -i inventory.ini playbooks/13-start-pbaas.yml -e pbaas_chain=vdex
Delegate: ansible-playbook -i inventory.ini playbooks/08-start-vrsc.yml
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|----------|---------|
| `15` | `15-sync-status.yml` | Check sync status (this procedure) |
| `13` | `13-start-pbaas.yml` | Start vDEX |
| `08` | `08-start-vrsc.yml` | Start VRSC |
| `14` | `14-shutdown-pbaas.yml` | Stop vDEX |
| `10` | `10-shutdown-vrsc.yml` | Stop VRSC |

---

## History

- **2026-05-07** — Created. 15-minute shortcut discovered after manually debugging MCP RPC auth issues while chain was perfectly healthy via `docker exec`.
