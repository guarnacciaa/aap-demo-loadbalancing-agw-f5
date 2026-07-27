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
| `{{ demo_inventory_name }}` (constructed parent) | Not tested | — | |
| `{{ demo_azure_inventory_name }}` | Not tested | — | |
| `{{ demo_aws_inventory_name }}` | Not tested | — | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| LB - Connectivity check (dry run) | Partial | 2026-07-24 | `lb_precheck_connectivity.yml` reaches the Azure/F5 backends and the AGW is confirmed reachable (`gateways` list populated), but the reachability assert referenced the wrong return key (`applicationgateways` instead of `gateways`) and failed — fixed this session. Not yet re-run to confirm the fix. |
| LB - Pool status preview (dry run) | Not tested | — | |
| LB - Verify VM in pool | Not tested | — | |
| LB - Verify connection status | Not tested | — | |
| LB - Drain and disconnect VM | Not tested | — | |
| LB - Reconnect VM to pool | Not tested | — | |
| LB - Collect results | Not tested | — | |
| Setup - Azure infrastructure | Not tested | — | |
| Setup - F5 pool member | Not tested | — | |
| Teardown - Azure infrastructure | Not tested | — | |
| Teardown - F5 pool member | Not tested | — | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Disconnect VM from load balancer | Not tested | — | Blocked on connectivity check node above until re-run confirms the fix |
| WF - Reconnect VM to load balancer | Not tested | — | Blocked on connectivity check node above until re-run confirms the fix |
| WF - Demo setup | Not tested | — | |
| WF - Demo teardown | Not tested | — | |

## Open issues

- None
