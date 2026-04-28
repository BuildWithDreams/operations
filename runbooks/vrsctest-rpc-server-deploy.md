# Deploy RPC Server on VRSCTEST

**Purpose:** Build and deploy the Rust Verus RPC server on the VRSCTEST testnet, providing a JSON-RPC API endpoint for external consumers.

**Why:** The VRSCTEST daemon's native RPC is only accessible from inside the container. The Rust RPC server proxies requests from external consumers to the daemon, handling authentication transparently.

**Chain:** VRSCTEST testnet
**Network:** `net-vrsctest` (`10.200.0.0/24`)
**Daemon:** `10.200.0.11:27486`
**RPC server:** `10.200.0.12:37486` → `127.0.0.1:37486`

---

## Prerequisites

- VRSCTEST daemon **running** (`08b-start-vrsctest.yml` complete, `docker ps` healthy)
- HITL approval obtained
- `rust_verusd_rpc_server` repo cloned on BWD (present at `/home/dream-hermes-agent/rust_verusd_rpc_server`)

---

## IP Addressing Convention

All chains use the same final-octet convention within their `/24` Docker network:

| Role | IP |
|---|---|
| Daemon | `.11` |
| RPC server | `.12` |

VRSC mainnet and VRSCTEST use identical octets on their respective networks — no IP conflict between them.

---

## Port Binding for Coexistence

If the VRSC mainnet RPC server is already running on `127.0.0.1:37486`, the VRSCTEST server must use a different host port (e.g. `37487`). The internal container port `37486` can remain the same — only the host port mapping changes.

Adjust `host_rpc_port` in `23b-rpc-server-deploy.yml` if coexistence is required.

---

## Procedure

### Step 1 — Configure Conf.toml

```
Delegate: ansible-playbook -i inventory.ini playbooks/21b-rpc-server-configure.yml
```

This playbook:
1. Reads daemon credentials from `vrsctest/data_dir/VRSCTEST.conf` (generated at first start)
2. Writes `Conf.toml` targeting `http://<user>:<pass>@10.200.0.11:27486`
3. Sets server bind to `10.200.0.12:37486`
4. Creates the multi-stage `Dockerfile`

> **Credential note:** If `VRSCTEST.conf` does not yet exist (first start), the playbook generates credentials. These must then be synced back to the daemon's `VRSCTEST.conf` before the RPC server can authenticate. If `getinfo` returns an auth error after deploy, the daemon was started before credentials were set — restart the daemon after configuring.

### Step 2 — Build Docker image

```
Delegate: ansible-playbook -i inventory.ini playbooks/22b-rpc-server-build.yml -e rpc_server_rebuild=true
```

> Use `-e rpc_server_rebuild=true` to force a clean build. Omit for idempotent skip if image already exists.

This playbook:
1. Touches `Conf.toml` to invalidate the Docker build cache layer
2. Runs `docker build --no-cache` with tag `buildwithdreams/rpc_server:vrsctest`

### Step 3 — Deploy container

```
Delegate: ansible-playbook -i inventory.ini playbooks/23b-rpc-server-deploy.yml
```

This playbook:
1. Removes any existing container (`dev200_rpc-rpc-1`)
2. Writes `docker-compose.yml` targeting `net-vrsctest`
3. Runs `docker compose up -d --force-recreate`
4. Waits for container to appear in `docker ps`

### Step 4 — Verify

```
Delegate: ansible-playbook -i inventory.ini playbooks/24b-rpc-server-getinfo.yml
```

Confirm:
- `getinfo` returns valid JSON with `result.blocks` ≥ 0
- `result.chain` = `VRSCTEST`
- No `"error"` field in response

To verify manually from the local machine:
```bash
curl -s -f -X POST 'http://localhost:37486/api/getinfo' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getinfo","params":[]}'
```

---

## Credential Handling

The RPC server's `Conf.toml` credentials **must match** the daemon's `VRSCTEST.conf` credentials. Mismatch causes the RPC server to return `{"error":{"code":-32603,"message":"Internal error"}}`.

**To read daemon credentials from inside the container:**
```bash
docker exec dev200-vrsctest-1 cat /root/.komodo/VRSCTEST/VRSCTEST.conf | grep -E '^rpc(user|password|port)'
```

If credentials need to be rotated, update both files and restart both the daemon and the RPC server.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (7) Connection refused` | Server not listening yet | Wait 10s, retry. Check `docker ps` shows container running |
| TOML parse error in logs | Conf.toml in image is stale | Re-run Step 2 with `-e rpc_server_rebuild=true` |
| `{"error":{"code":-32603,"message":"Internal error"}}` | Credential mismatch | Sync credentials between Conf.toml and VRSCTEST.conf |
| Container keeps restarting | Conf.toml error or missing env | `docker logs dev200_rpc-rpc-1`, fix Conf.toml, rebuild |
| `rc=0` but empty response | Daemon not fully started | Wait for daemon to load block index, retry |

**Container restart loop:** If the container won't stay up:
```bash
docker rm -f dev200_rpc-rpc-1
# Then re-run Step 2 and Step 3
```

---

## Rollback

To remove the RPC server container:

```bash
ssh bwd "docker rm -f dev200_rpc-rpc-1"
```

To stop the container but preserve configuration:

```bash
ssh bwd "docker compose -f /home/dream-hermes-agent/rust_verusd_rpc_server/docker-compose.yml -p dev200_rpc stop"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|---|---|
| `21b` | `21b-rpc-server-configure.yml` | Write Conf.toml and Dockerfile |
| `22b` | `22b-rpc-server-build.yml` | Build Docker image |
| `23b` | `23b-rpc-server-deploy.yml` | Deploy container |
| `24b` | `24b-rpc-server-getinfo.yml` | Health check |
| `08b` | `08b-start-vrsctest.yml` | Start VRSCTEST daemon (prereq) |

---

## History

- **2026-04-28** — Created. Mirrors the VRSC mainnet RPC server deployment procedure with VRSCTEST-specific paths, IPs, and credential file.
