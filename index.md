---
layout: default
title: Home
nav_order: 1
---

# BWD Operations

Living operations documentation for BuildWithDreams infrastructure — playbooks, procedures, and runbooks.

{: .info }
> This is a living document. Procedures are updated as infrastructure evolves.

## Quick Links

- [Runbooks](runbooks/) — Step-by-step operational procedures
- [Playbooks](playbooks/) — Ansible playbook reference
- [Infrastructure](infrastructure/) — Network, containers, and services

## Core Principles

### Playbook-First Operations
Every operational change to BWD infrastructure must go through a playbook. No raw SSH unless no playbook exists for that task.

**Workflow:**
1. Check `~/provisioning/playbooks/` for an existing playbook
2. If none exists: create a GitHub issue, get HITL consent, then implement
3. If one exists: delegate execution via the executor pattern

### HITL Oversight
Human-in-the-loop approval is required before:
- Any destructive operation (stopping chains, deleting data)
- Any config change to running daemons
- Any new playbook or script that will run on BWD
- Any commit to infrastructure repositories

### Delegation Pattern
Remote operations use a **planner + executor** pattern:

```
Operator request
    ↓
Planner: checks playbooks, plans approach
    ↓
  [No playbook?] → Flag gap, create issue, wait for consent
    ↓ [Playbook exists]
Executor (role='leaf', toolsets=['terminal'] only)
  → Runs ansible-playbook command
  → Returns PLAY RECAP
    ↓
Planner: surfaces results to operator
```
