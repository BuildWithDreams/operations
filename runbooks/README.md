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
| [Add rpcallowip for VRSCTEST](runbooks/vrsctest-rpcallowip.html) | Add Docker network subnet to vRSCTEST RPC allow list | No |
| [Restart a chain](runbooks/chain-restart.html) | Graceful stop + start for any chain | No |
| [Check sync status](runbooks/sync-status.html) | Verify chain is fully synced | No |
| [Clean chainstate](runbooks/chain-clean.html) | Wipe chainstate for fresh resync | No |

## Deployment

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Deploy RPC server](runbooks/rpc-server-deploy.html) | Build and deploy Rust RPC server | No |
| [Deploy RVT SPA](runbooks/rvt-deploy.html) | Deploy RVT single-page app | No |

## Emergency

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Container failed to start](runbooks/container-fails.html) | Debug and recover a crashed container | No |
