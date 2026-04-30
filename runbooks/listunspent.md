---
layout: default
title: Query wallet UTXOs via listunspent
nav_order: 1
parent: Runbooks
---

# Query wallet UTXOs via listunspent

**Purpose:** Retrieve unspent transaction outputs (UTXOs) from a chain's wallet using the `listunspent` RPC method via the `verus` CLI.

**Why:** The raw UTXO set is needed for wallet analysis, coin selection, or building transactions. Filtering and parsing can be layered on later.

**Chains:** VRSC, vDEX (mainnet PBaaS), VRSCTEST

---

## Prerequisites

- Container for the target chain is **running**
- Wallet is **unlocked** (listunspent only returns outputs the wallet can spend)
- HITL approval obtained

---

## Procedure

### Option A — VRSCTEST

```
Delegate: ansible-playbook -i inventory.ini playbooks/25b-listunspent-vrsctest.yml
```

### Option B — VRSC and vDEX (mainnet)

```
Delegate: ansible-playbook -i inventory.ini playbooks/25-listunspent.yml
```

---

## Output

The playbook prints the raw JSON array returned by `listunspent` for each chain:

```
VRSCTEST:
[
  {
    "txid": "7a663b77b9cc84789672f7f2a7f9006207343bf71bc64daa5e50642e740467cf",
    "vout": 0,
    "generated": false,
    "address": "RFdiZoMDDUUYfoucW1zzpGsRhochG7RLUu",
    "account": "",
    "amount": 300.00000000,
    "scriptPubKey": "76a91445b2861471096b9ffb7957c6676004e041c2f11a88ac",
    "confirmations": 436,
    "spendable": true
  }
]
```

**Field reference:**

| Field | Description |
|-------|-------------|
| `txid` | Transaction ID |
| `vout` | Output index within the transaction |
| `address` | The VRSC address receiving the output |
| `amount` | Denomination in the chain's native token |
| `confirmations` | Block confirmations |
| `spendable` | Whether the wallet can spend this output |
| `generated` | Whether the output was from coinbase/mining |
| `scriptPubKey` | Raw script hex |

---

## Filtering (not yet implemented)

Future version will support `-e` flags to filter by confirmation range:

```bash
# UTXOs with 0 confirmations (mempool)
ansible-playbook -i inventory.ini playbooks/25-listunspent.yml -e "listunspent_minconf=0" -e "listunspent_maxconf=0"

# UTXOs with 1–6 confirmations (recently confirmed)
ansible-playbook -i inventory.ini playbooks/25-listunspent.yml -e "listunspent_minconf=1" -e "listunspent_maxconf=6"
```

---

## Relevant Playbooks

| # | Playbook | Purpose |
|---|----------|---------|
| `25` | `25-listunspent.yml` | VRSC + vDEX UTXO query |
| `25b` | `25b-listunspent-vrsctest.yml` | VRSCTEST UTXO query |
| `15b` | `15b-sync-status-vrsctest.yml` | Check VRSCTEST sync status |

---

## History

- **2026-04-30** — Created. Initial version returns raw `listunspent` output with no filtering.
