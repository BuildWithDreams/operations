---
layout: default
title: Set SOURCE_OF_FUNDS for svc-idcreate
nav_order: 4
parent: Deployment
---

# Set SOURCE_OF_FUNDS for svc-idcreate

**Purpose:** Set the `SOURCE_OF_FUNDS` R-address in the `svc-idcreate` `.env` and force-restart the running containers so the new value is picked up at runtime.

**Why:** The identity creation service uses a funding R-address to generate Z-addresses for identity registrations. The R-address must be set before the service can function correctly.

**Chain:** VRSCTEST testnet
**Compose project:** `dev200_idcreate`
**Service IP:** `10.200.0.14`

---

## Prerequisites

- `svc-idcreate` deployed (`41-idcreate-deploy.yml` run successfully)
- R-address obtained from the operator
- HITL approval obtained

---

## Procedure

### Step 1 — Set SOURCE_OF_FUNDS and restart containers

```bash
Delegate: ansible-playbook -i inventory.ini playbooks/44-idcreate-source-of-funds.yml \
  -e "source_of_funds_address=RXXXXXXXXXXXXXXXXXXXXXXXX"
```

Replace `RXXXXXXXXXXXXXXXXXXXXXXXX` with the actual R-address provided by the operator.

This playbook:
1. Validates that `source_of_funds_address` is provided
2. Checks that `~/svc-idcreate/.env` exists (fails if `41` has not been run)
3. Writes `SOURCE_OF_FUNDS="<address>"` to `.env` using `lineinfile` (idempotent — replaces existing value)
4. Runs `docker compose -p dev200_idcreate up -d --force-recreate` to force containers to re-read `.env`
5. Waits for both `api-1` and `worker-1` containers to be `Up`

> **Why `--force-recreate` and not `restart`?** `docker compose restart` does **not** re-read `.env` — environment is baked in at `up` time. The `--force-recreate` flag stops the old containers and starts new ones, causing Docker Compose to re-read the updated `.env` file.

### Step 2 — Verify

```bash
ssh bwd "docker logs dev200_idcreate-api-1 --tail 20"
ssh bwd "docker logs dev200_idcreate-worker-1 --tail 20"
```

Confirm the containers started cleanly with no errors referencing the R-address.

---

## What to check if something goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| Playbook fails: `.env not found` | `41-idcreate-deploy.yml` not run yet | Run playbook `41` first |
| Playbook fails: `source_of_funds_address is required` | Missing `-e` argument | Provide the R-address with `-e "source_of_funds_address=Rxxx..."` |
| Containers not running after playbook | Docker failed to start containers | `ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate up -d"` |
| Old SOURCE_OF_FUNDS value still in use | Containers not recreated | Re-run the playbook — it is idempotent and will recreate |

---

## Rollback

To update with a different R-address, simply re-run the playbook with the new address:

```bash
Delegate: ansible-playbook -i inventory.ini playbooks/44-idcreate-source-of-funds.yml \
  -e "source_of_funds_address=Ryyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

The old value is replaced automatically.

To remove the value entirely (not recommended — service will not function):

```bash
ssh bwd "grep SOURCE_OF_FUNDS ~/svc-idcreate/.env"
ssh bwd "sed -i '/^SOURCE_OF_FUNDS=/d' ~/svc-idcreate/.env"
ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate up -d --force-recreate"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `41` | `41-idcreate-deploy.yml` | Deploy idcreate stack; creates `.env` |
| `43` | `43-idcreate-add-api-key.yml` | Generate `REGISTRAR_API_KEYS` (same pattern as this playbook) |
| `44` | `44-idcreate-source-of-funds.yml` | Set `SOURCE_OF_FUNDS` + restart |

---

## History

- **2026-04-30** — Created. Mirrors playbook `43-idcreate-add-api-key.yml` pattern. Uses `lineinfile` for idempotent `.env` updates and `docker compose up -d --force-recreate` to ensure containers pick up the new value.
