# Testing log

Tracks testing progress for this demo. Update after each session. For procedural verification steps, see [verification.md](verification.md).

**Status values:** `Not tested` · `Pass` · `Partial` · `Fail` · `Blocked`

## Status summary

### Entry playbooks

| Component | Status | Last tested | Notes |
|---|---|---|---|
| CasC apply (`aap_config.yml`) | Not tested | — | |
| CasC cleanup (`aap_cleanup.yml`) | Not tested | — | |
| Smoke test (`verify.yml`) | Not tested | — | |

### Execution environments

| Component | Status | Last tested | Notes |
|---|---|---|---|
| EE build and push | Not tested | — | |

### Inventories

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Parent inventory (`demo_inventory_name`) | Not tested | — | |
| Azure child inventory (`demo_azure_inventory_name`) | Not tested | — | |
| AWS child inventory (`demo_aws_inventory_name`, reserved/unused in UC1) | Not tested | — | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| LB - Connectivity check (dry run) | Not tested | — | |
| LB - Pool status preview (dry run) | Not tested | — | |
| LB - Verify VM in pool | Not tested | — | |
| LB - Verify connection status | Not tested | — | |
| LB - Drain and disconnect VM | Not tested | — | |
| LB - Reconnect VM to pool | Not tested | — | |
| LB - Collect results | Not tested | — | |
| Setup - Azure infrastructure (infra, `demo_manage_infrastructure: true`) | Not tested | — | |
| Setup - F5 pool member (infra, `demo_manage_infrastructure: true`) | Not tested | — | |
| Teardown - Azure infrastructure (infra, `demo_manage_infrastructure: true`) | Not tested | — | |
| Teardown - F5 pool member (infra, `demo_manage_infrastructure: true`) | Not tested | — | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Disconnect VM from load balancer | Not tested | — | |
| WF - Reconnect VM to load balancer | Not tested | — | |
| WF - Demo setup (infra, `demo_manage_infrastructure: true`) | Not tested | — | |
| WF - Demo teardown (infra, `demo_manage_infrastructure: true`) | Not tested | — | |

## Open issues

- None
