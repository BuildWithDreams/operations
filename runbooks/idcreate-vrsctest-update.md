---
layout: default
title: Update svc-idcreate on VRSCTEST
nav_order: 4
parent: Deployment
---

# Update svc-idcreate on VRSCTEST

**Purpose:** Pull latest commits from `BuildWithDreams/svc-idcreate`, rebuild the Docker image, and restart the idcreate stack on VRSCTEST.

**When to use:** New commits landed in the `svc-idcreate` repo that need to be deployed to BWD.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Service IP:** `10.200.0.14`
**URL:** `https://idcreate.vrsctest.buildwithdreams.com`

---

## Prerequisites

- svc-idcreate already deployed (`idcreate-vrsctest-deploy.md` complete)
- VRSCTEST daemon running
- HITL approval obtained

---

## Procedure

### Step 1 — Run the update playbook

```
Delegate: ansible-playbook -i inventory.ini playbooks/45-idcreate-update.yml
```

This playbook (in order):
1. **Backs up** `~/.env` → `~/.env.backup` (before any git operation)
2. **Pulls** latest commits from `BuildWithDreams/svc-idcreate`
3. **Restores** `.env` from backup (git merge/reset can remove untracked files)
4. **Stops** running containers
5. **Rebuilds** `buildwithdreams/svc-idcreate:local` from latest code
6. **Starts** containers with `docker compose up -d --force-recreate`
7. **Verifies** api container appears in `docker ps`

> `.env` is restored unconditionally after every pull. This is required because git operations can remove or replace untracked files during merge/rebase.

### Step 2 — Verify

```bash
curl -s http://10.200.0.14:5003/health | python3 -c "import sys,json; d=json.load(sys.stdin); print('status:', d['status'], '| native_coin:', d.get('native_coin'), '| rpc:', d['rpc']['reachable'])"
```

Expected: `status: ok | native_coin: None | rpc: True`

To verify from the server:
```bash
ssh bwd "docker ps | grep dev200_idcreate"
ssh bwd "curl -s http://10.200.0.14:5003/health"
```

### Step 3 — Check logs if anything looks wrong

```bash
ssh bwd "docker logs dev200_idcreate-api-1 --tail 30"
ssh bwd "docker logs dev200_idcreate-worker-1 --tail 30"
```

---

## .env preservation

The playbook backs up `.env` before every pull. The backup lives at:

```
~/svc-idcreate/.env.backup
```

If anything goes wrong with the `.env` restore step, manually restore from backup:

```bash
ssh bwd "cp ~/svc-idcreate/.env.backup ~/svc-idcreate/.env && cd ~/svc-idcreate && docker compose -p dev200_idcreate up -d --force-recreate"
```

---

## If the container doesn't come up

Common causes:

| Symptom | Check | Fix |
|---|---|---|
| Container not starting | `docker logs dev200_idcreate-api-1` | Check for import errors or missing env vars |
| RPC unreachable | `docker logs dev200_idcreate-api-1` | Verify VRSCTEST daemon at `10.200.0.11:18843` is running |
| .env missing | `ls ~/svc-idcreate/.env*` on server | `cp ~/.env.backup ~/.env` then restart |
| Image missing | `docker images | grep svc-idcreate` | Run `40-idcreate-build.yml` first |

Full redeploy if needed:
```
Delegate: ansible-playbook -i inventory.ini playbooks/41-idcreate-deploy.yml
```

---

## Rollback

To roll back to the previous container state (using whatever image was previously built):

```bash
ssh bwd "cd ~/svc-idcreate && docker compose -p dev200_idcreate restart"
```

To roll back to a specific prior commit:

```bash
# Identify the last good commit
ssh bwd "cd ~/svc-idcreate && git log --oneline -5"

# Reset to it
ssh bwd "cd ~/svc-idcreate && git reset --hard <commit-hash>"

# Rebuild and restart
Delegate: ansible-playbook -i inventory.ini playbooks/45-idcreate-update.yml
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `45` | `45-idcreate-update.yml` | Pull + rebuild + restart idcreate stack |
| `40` | `40-idcreate-build.yml` | Build image (used by 45, rarely needed alone) |
| `41` | `41-idcreate-deploy.yml` | Full deploy — writes `.env` fresh |

---

## History

- **2026-04-30** — Created. Pull/rebuild/restart pattern for svc-idcreate on VRSCTEST. `.env` is backed up before pull and restored unconditionally after to survive git merge side-effects.
