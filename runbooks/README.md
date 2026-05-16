---
layout: default
title: Runbooks
nav_order: 2
---

# Runbooks

Step-by-step operational procedures for BWD infrastructure.

{: .warning }
> All runbooks require HITL approval before execution unless marked **Automated**.

---

## Chain Operations

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Add rpcallowip for VRSCTEST](vrsctest-rpcallowip/) | Add Docker network subnet to vRSCTEST RPC allow list | No |
| [Restart a chain](chain-restart/) | Graceful stop + start for any chain | No |
| [Check sync status](sync-status/) | Verify chain is fully synced | No |
| [Clean chainstate](chain-clean/) | Wipe chainstate for fresh resync | No |

## Deployment

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Deploy RPC Server on VRSCTEST](vrsctest-rpc-server-deploy/) | Build and deploy Rust RPC server on VRSCTEST testnet | No |
| [Deploy QRcodes on VRSCTEST](qrcodes-vrsctest-deploy/) | Build and deploy QRcode creator on VRSCTEST testnet | No |
| [Deploy idcreate on VRSCTEST](idcreate-vrsctest-deploy/) | Build and deploy Verus identity creation service on VRSCTEST testnet | No |
| [Deploy idcreate on VRSC](idcreate-vrsc-deploy/) | Build and deploy Verus identity creation service on VRSC mainnet | No |
| [Deploy RPC Server](rpc-server-deploy/) | Build and deploy Rust RPC server (VRSC mainnet) | No |
| [Deploy RVT SPA](rvt-deploy/) | Deploy RVT single-page app | No |

## Conventions

Operational standards and patterns for all playbooks and runbooks:

| Convention | Description |
|---|---|
| [Caddyfile management](idcreate-vrsc-deploy/#caddyfile-management) | Host file edits + validate/reload (no docker cp, no fmt inside container) |
| [Route idempotency checks](idcreate-vrsc-deploy/#route-idempotency-checks) | Use `grep -Fq` for exact-domain matching |
| [Variable scoping](idcreate-vrsc-deploy/#variable-scoping) | Chain-prefixed vars (`idcreate_vrsc_*`) vs shared defaults |
| [Docker network routing](idcreate-vrsc-deploy/#docker-network-routing) | PBaaS chains isolated by network; Caddy must be multi-homed |
| [Compose project naming](idcreate-vrsc-deploy/#compose-project-naming) | Use `COMPOSE_PROJECT_NAME` env var, not hardcoded `container_name` |

## Emergency

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Container failed to start](container-fails/) | Debug and recover a crashed container | No |
