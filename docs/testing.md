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
| LB - Connectivity check (dry run) | Not tested | — | Read-only; checks Azure AGW reachability and generic F5 pool-collection reachability (no fixed pool) |
| LB - Pool status preview (dry run) | Not tested | — | Read-only; previews AGW NIC membership and every discovered F5 pool membership |
| LB - Verify VM in pool | Not tested | — | AGW and F5 paths; F5 path asserts at least one discovered pool membership |
| LB - Verify connection status | Not tested | — | F5 path reports enabled/disabled per discovered pool |
| LB - Drain and disconnect VM | Not tested | — | F5 path loops over every discovered pool membership |
| LB - Reconnect VM to pool | Not tested | — | F5 path loops over every discovered pool membership |
| LB - Collect results | Not tested | — | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup | Not tested | — | Chains Azure + F5 setup |
| WF - Demo teardown | Not tested | — | Chains F5 + Azure teardown |
| WF - Disconnect VM from load balancer | Not tested | — | AGW and F5 paths |
| WF - Reconnect VM to load balancer | Not tested | — | AGW and F5 paths |

## Open issues

- Not yet tested end-to-end: the F5 side of every `LB - *` scenario/dry-run template now
  discovers pool, partition, and member port from the target VM's IP at runtime
  (`playbooks/demo/tasks/f5_discover_pool_from_ip.yml`) instead of reading fixed
  `f5_pool_name`/`f5_partition`/`f5_pool_member_port` variables — see
  [docs/setup.md#f5-pool-and-partition-discovery](setup.md#f5-pool-and-partition-discovery).
  `f5_pool_name`/`f5_partition`/`f5_pool_member_port` are no longer passed via `extra_vars` to
  any scenario template (see `group_vars/all/job_templates.yml`); they remain required only
  for `Setup - F5 pool member` / `Teardown - F5 pool member`. Needs an end-to-end F5 run
  against a node that is a member of one pool, and ideally a second run against a node that
  is a member of multiple pools/partitions at once, to confirm discovery finds and acts on
  every membership correctly.
