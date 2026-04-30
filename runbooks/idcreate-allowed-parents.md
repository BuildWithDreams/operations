---
layout: default
title: Set REGISTRAR_ALLOWED_PARENTS for svc-idcreate
nav_order: 5
parent: Deployment
---

# Set REGISTRAR_ALLOWED_PARENTS for svc-idcreate

**Purpose:** Set the `REGISTRAR_ALLOWED_PARENTS` currency ID list in the `svc-idcreate` `.env` and force-restart the running containers so the new value is picked up at runtime.

**Why:** The identity creation service needs to know which Verus chain currencies are valid parents for identities it registers. This list must be set before the service can function correctly.

**Chain:** VRSCTEST testnet
**Compose project:** `dev200_idcreate`
**Service IP:** `10.200.0.14`

---

## Prerequisites

- `svc-idcreate` deployed (`41-idcreate-deploy.yml` run successfully)
- Currency ID(s) obtained from the operator
- HITL approval obtained

---

## Procedure

### Step 1 — Set REGISTRAR_ALLOWED_PARENTS and restart containers

```bash
Delegate: ansible-playbook -i inventory.ini playbooks/46-idcreate-allowed-parents.yml \
  -e "currency_ids=VRSC,RVSR"
```

Replace `VRSC,RVSR` with the actual comma-separated currency ID list provided by the operator. On VRSC mainnet these are names like `VRSC`, `RVSR`. On PBaaS chains they are hex string IDs.

This playbook:
1. Validates that `currency_ids` is provided
2. Checks that `~/svc-idcreate/.env` exists (fails if `41` has not been run)
3. Writes `REGISTRAR_ALLOWED_PARENTS="<ids>"` to `.env` using `lineinfile` (idempotent — replaces existing value)
4. Runs `docker compose -p dev200_idcreate up -d --force-recreate` to force containers to re-read `.env`
5. Waits for both `api-1` and `worker-1` containers to be `Up`

> **Why `--force-recreate` and not `restart`?** `docker compose restart` does **not** re-read `.env` — environment is baked in at `up` time. The `--force-recreate` flag stops the old containers and starts new ones, causing Docker Compose to re-read the updated `.env` file.

### Step 2 — Verify

```bash
ssh bwd "docker logs dev200_idcreate-api-1 --tail 20"
ssh bwd "docker logs dev200_idcreate-worker-1 --tail 20"
```

Confirm the containers started cleanly with no errors referencing the currency IDs.

---

## What to check if something goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| Playbook fails: `.env not found` | `41-idcreate-deploy.yml` not run yet | Run playbook `41` first |
| Playbook fails: `currency_ids is required` | Missing `-e` argument | Provide the currency IDs with `-e "currency_ids=VRSC,RVSR"` |
| Containers not running after playbook | Docker failed to start containers | `ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate up -d"` |
| Old REGISTRAR_ALLOWED_PARENTS value still in use | Containers not recreated | Re-run the playbook — it is idempotent and will recreate |

---

## Rollback

To update with a different currency ID list, simply re-run the playbook with the new list:

```bash
Delegate: ansible-playbook -i inventory.ini playbooks/46-idcreate-allowed-parents.yml \
  -e "currency_ids=VRSC,RVSR,this"
```

The old value is replaced automatically.

To remove the value entirely (not recommended — service will not function):

```bash
ssh bwd "grep REGISTRAR_ALLOWED_PARENTS ~/svc-idcreate/.env"
ssh bwd "sed -i '/^REGISTRAR_ALLOWED_PARENTS=/d' ~/svc-idcreate/.env"
ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate up -d --force-recreate"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `41` | `41-idcreate-deploy.yml` | Deploy idcreate stack; creates `.env` |
| `43` | `43-idcreate-add-api-key.yml` | Generate `REGISTRAR_API_KEYS` |
| `44` | `44-idcreate-source-of-funds.yml` | Set `SOURCE_OF_FUNDS` |
| `46` | `46-idcreate-allowed-parents.yml` | Set `REGISTRAR_ALLOWED_PARENTS` + restart |

---

## History

- **2026-04-30** — Created. Mirrors playbook `44-idcreate-source-of-funds.yml` pattern. Uses `lineinfile` for idempotent `.env` updates and `docker compose up -d --force-recreate` to ensure containers pick up the new value.
