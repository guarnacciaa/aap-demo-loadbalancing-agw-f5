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
| LB - Connectivity check (dry run) | Partial | 2026-07-24 | `lb_precheck_connectivity.yml` reaches the Azure/F5 backends and the AGW is confirmed reachable (`gateways` list populated), but the reachability assert referenced the wrong return key (`applicationgateways` instead of `gateways`) and failed — fixed 2026-07-24. Its F5 branch also goes through `tasks/f5_authenticate.yml`, so it is affected by the two 2026-07-28 fixes below. Not yet re-run to confirm either fix. |
| LB - Pool status preview (dry run) | Blocked | 2026-07-28 | `lb_pool_preview.yml` failed authenticating to F5: (1) `no_log` was nested inside the `ansible.builtin.uri` module args instead of at task level, causing `uri` to reject it as an unsupported parameter — fixed. (2) After that fix, the real auth failure was still hidden by `no_log`'s censoring of the whole result on error — added a `block`/`rescue` in `tasks/f5_authenticate.yml` to surface a credential-safe status/msg diagnostic instead. (3) No F5 management-port handling existed anywhere in the demo; added `tasks/f5_resolve_management_port.yml` (probes 443, falls back to 8443 — the port BIG-IP VE marketplace/lab images commonly use) and wired `f5_resolved_management_port` through every raw `uri` call and every `bigip_pool_member` `provider.server_port`. Also fixed `f5_credential_name` (was the generic `F5 BIG-IP API`, not artifact-unique). None of these fixes has been re-run against the real F5 yet — still blocked on confirming which port/cause was the actual root cause. |
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

- F5 authentication is not yet confirmed working end-to-end. Three fixes landed 2026-07-28 (`no_log` placement, credential-safe error surfacing, F5 management-port auto-detection) but none has been validated against the real F5 instance yet — re-run `LB - Pool status preview (dry run)` (or `lb_precheck_connectivity.yml`) next session and update this log with the actual result, including which management port (443 or 8443) the auto-detection resolved to.
