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
| LB - Drain and disconnect VM | Partial | 2026-07-28 | NIC detach confirmed working: `agw_membership_update.yml`'s `azure_rm_networkinterface` task failed with `(InvalidResourceReference)` on the subnet because `virtual_network` was passed as a bare name — the module then assumes the VNet is in the same resource group as the NIC, but here it lives in a separate networking resource group. Fixed by passing `virtual_network` as a `{name, resource_group}` dict sourced from `azure_rm_networkinterface_info`'s return; re-run confirmed `changed: true` with no error. The same job run then failed later, in `lb_drain_disconnect.yml`'s "Wait for AGW connection drain window" task, with `recursive loop detected in template string: {{ drain_poll_interval \| default(15) \| int }}` — a self-referencing play var that only resolves when supplied via extra_vars. Fixed by moving `\| default(...)` to the point of use, and by replacing the same fragile pattern for the required `lb_backend_type`/`target_vm_hostname` vars with an explicit `pre_tasks` assert (applied consistently across all 6 `lb_*.yml` playbooks: `lb_drain_disconnect.yml`, `lb_reconnect.yml`, `lb_verify_presence.yml`, `lb_verify_status.yml`, `lb_pool_preview.yml`, `lb_precheck_connectivity.yml`). A further bug found 2026-07-28 while investigating the reconnect failure below: `agw_membership_update.yml`'s "Persist the removed..." task wrote the state tag using a YAML mapping with a templated key (`"{{ agw_state_tag_key }}": ...`) — Ansible does not render Jinja expressions used as YAML mapping keys in module args, so every run persisted the tag under the literal string `{{ agw_state_tag_key }}` instead of `lb_agw_backend_pool_id`, silently breaking the save/restore cycle. Fixed by building the whole `tags` dict as a single Jinja expression (`tags: "{{ {agw_state_tag_key: ...} }}"`). Not yet re-run end-to-end to confirm either the drain-wait fix or the tag-key fix. |
| LB - Reconnect VM to pool | Blocked | 2026-07-28 | Shares `agw_membership_update.yml` with the row above, so benefits from the same `virtual_network` fix. The re-attach path was separately reviewed and found already correct: the saved AGW backend pool tag stores a full resource ID, which the module uses verbatim without reconstructing it from name + resource group, so it is not affected by the same class of bug. Also received the `lb_backend_type`/`target_vm_hostname` pre_tasks assert fix (see row above). First real run failed as expected with "Fail if no saved Application Gateway backend pool association exists to restore" because the VM's tags were empty (`{}`) — root-caused to the tag-key templating bug described in the row above (the save never actually landed under the correct key in any prior drain run). Fixed there; needs a drain run to persist a correctly-keyed tag before this can be re-tested. |
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
- AGW drain/disconnect is not yet confirmed working end-to-end. Three fixes landed 2026-07-28 (`azure_rm_networkinterface`'s `virtual_network` resource-group handling; the `drain_poll_interval`/`lb_backend_type`/`target_vm_hostname` self-referencing var recursion; the `agw_state_tag_key` templated-dict-key bug that broke tag persistence) — re-run `LB - Drain and disconnect VM` next session and update this log with the actual result, including whether the drain-wait pause now completes without error and whether the VM ends up with a `lb_agw_backend_pool_id` tag (correct key) holding the pool's resource ID.
- Before the next `LB - Reconnect VM to pool` test, check the VM's tags in the Azure portal for a leftover `{{ agw_state_tag_key }}` literal-key tag from earlier buggy runs and remove it manually — it is dead data now that the key is fixed, and would otherwise sit alongside the correctly-keyed tag indefinitely.
- Once a drain run persists a correctly-keyed tag, re-run `LB - Reconnect VM to pool` and confirm it finds `agw_discovered_pool_id_saved` populated and successfully re-attaches the NIC.
