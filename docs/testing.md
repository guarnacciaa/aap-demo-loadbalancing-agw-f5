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
| EE build (`ee-loadbalancing-agw-f5`) | Not tested | — | Requires `ansible-builder` and PAH access |
| EE push to PAH | Not tested | — | |

### Inventories

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Demo-LoadBalancingAGWF5 (constructed parent) | Not tested | — | |
| Azure-Resources-LoadBalancingAGWF5 (child) | Not tested | — | |
| AWS-Resources-LoadBalancingAGWF5 (child — reserved, unused in UC1) | Not tested | — | |

### Setup and teardown job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Setup - Azure infrastructure | Not tested | — | Dual-mode: BYO and create-from-scratch |
| Setup - F5 pool member | Not tested | — | Requires BYO F5 BIG-IP |
| Teardown - Azure infrastructure | Not tested | — | |
| Teardown - F5 pool member | Not tested | — | |

### Demo job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| LB - Connectivity check (dry run) | Not tested | — | Read-only; checks Azure AGW and F5 API reachability |
| LB - Pool status preview (dry run) | Not tested | — | Read-only; previews AGW/F5 pool membership |
| LB - Verify VM in pool | Not tested | — | AGW and F5 paths |
| LB - Verify connection status | Not tested | — | |
| LB - Drain and disconnect VM | Not tested | — | |
| LB - Reconnect VM to pool | Not tested | — | |
| LB - Collect results | Not tested | — | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup | Not tested | — | Chains Azure + F5 setup |
| WF - Demo teardown | Not tested | — | Chains F5 + Azure teardown |
| WF - Disconnect VM from load balancer | Not tested | — | AGW and F5 paths |
| WF - Reconnect VM to load balancer | Not tested | — | AGW and F5 paths |

## Open issues

- None
