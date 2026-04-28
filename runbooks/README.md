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
| [Deploy RPC Server](rpc-server-deploy/) | Build and deploy Rust RPC server (VRSC mainnet) | No |
| [Deploy RVT SPA](rvt-deploy/) | Deploy RVT single-page app | No |

## Emergency

| Procedure | Description | Automated |
|-----------|-------------|-----------|
| [Container failed to start](container-fails/) | Debug and recover a crashed container | No |
