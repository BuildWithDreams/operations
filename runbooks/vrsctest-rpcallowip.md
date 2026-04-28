# Add rpcallowip for VRSCTEST Docker Network

**Purpose:** Allow applications running in containers on the `net-vrsctest` Docker network to connect to the vRSCTEST daemon's RPC port.

**Why:** The daemon's default `rpcallowip=127.0.0.1` only permits local host connections. Any app in a container on `net-vrsctest` (e.g. at `10.200.0.x`) will be rejected.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Config:** `vrsctest/data_dir/vrsctest.conf`

---

## Prerequisites

- Container is **running**
- HITL approval obtained

---

## Procedure

### Step 1 — Stop vRSCTEST

```
Delegate: ansible-playbook -i inventory.ini playbooks/10b-shutdown-vrsctest.yml
```

> The daemon reads config at startup. Changes to the config file while it's running have no effect until the daemon is restarted.

### Step 2 — Add rpcallowip entry

```
Delegate: ansible-playbook -i inventory.ini playbooks/16b-add-vrsctest-rpc-allowip.yml
```

This playbook:
1. Inspects `net-vrsctest` to detect the actual subnet
2. Appends `rpcallowip=<subnet>/<cidr>` to `vrsctest/data_dir/vrsctest.conf`
3. Is **idempotent** — safe to run multiple times; only adds the entry if not already present

### Step 3 — Start vRSCTEST

```
Delegate: ansible-playbook -i inventory.ini playbooks/08b-start-vrsctest.yml
```

### Step 4 — Verify

```
Delegate: ansible-playbook -i inventory.ini playbooks/15b-sync-status-vrsctest.yml
```

Confirm:
- `rpcallowip` in config includes `10.200.0.0/24`
- Node is synced (`longestchain` equals block height)
- `connections` > 0 (connected to peers)

---

## Rollback

To remove the `rpcallowip` entry after approval:

```bash
ssh bwd "sudo sed -i '/^rpcallowip=10.200.0.0/d' /home/dream-hermes-agent/docker-verusd/vrsctest/data_dir/vrsctest.conf"
```

Then restart: `Delegate: ansible-playbook -i inventory.ini playbooks/08b-start-vrsctest.yml`

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|----------|---------|
| `10b` | `10b-shutdown-vrsctest.yml` | Stop vRSCTEST |
| `08b` | `08b-start-vrsctest.yml` | Start vRSCTEST |
| `16b` | `16b-add-vrsctest-rpc-allowip.yml` | Add rpcallowip (this procedure) |
| `15b` | `15b-sync-status-vrsctest.yml` | Check sync status |

---

## History

- **2026-04-28** — Created. Volume mount was also broken (data not persisting). Both fixed in single session.
