# Demo procedures

## Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml -i inventory.yml --vault-id @prompt
```

## Workflow 1 — Disconnect VM

1. Launch **WF - Disconnect VM from load balancer**.
2. Survey: `lb_backend_type` = `agw` or `f5`, target hostname (target IP is resolved automatically — see [docs/setup.md#automatic-private-ip-discovery](setup.md#automatic-private-ip-discovery) — unless you provide an explicit override).
3. Observe nodes: connectivity check → pool status preview → verify presence → verify status → drain/disconnect → collect results.

## Workflow 2 — Reconnect VM

1. Launch **WF - Reconnect VM to load balancer**.
2. Provide the same survey values used for disconnect.
3. Observe nodes: connectivity check → pool status preview → reconnect → verify status → collect results.

## Standalone job template launches (AAP UI)

Every `LB - *` job template can be launched directly from AAP **Templates**, outside either workflow. Each one also has its own survey (mirroring the workflow survey, scoped to that template — see `group_vars/all/job_templates.yml`), so launching from the UI prompts for `lb_backend_type` and `target_vm_hostname` (and, where applicable, an optional `target_vm_ip` override) with the AGW defaults pre-filled. Leaving a question unanswered keeps the template's own `extra_vars` default (`lb_backend_type: agw`, `target_vm_hostname: agw_vm_hostname`); no question is `required`, so the launch never fails even without touching the survey. To exercise the F5 VM standalone, either answer the survey with `lb_backend_type=f5` / `target_vm_hostname=<F5 VM hostname>`, or pass the equivalent CLI `-e` flags shown below (both take precedence over the defaults).

## Dry run checks (optional)

Read-only; never create, modify, or delete any resource. Run these before either
workflow above to confirm credentials and current pool state, or rely on the
connectivity check and pool preview nodes that already run automatically as the
first two nodes of both workflows.

```bash
# Confirm Azure (via the target VM/NIC) and F5 BIG-IP API connectivity
ansible-playbook playbooks/demo/lb_precheck_connectivity.yml \
  -e target_vm_hostname=azure-vm-agw01 \
  -e azure_resource_group=my-rg \
  --vault-id @prompt

# Preview current AGW/F5 pool membership for a target VM
# target_vm_ip is not needed: it is resolved automatically from
# target_vm_hostname's NIC (see docs/setup.md#automatic-private-ip-discovery).
ansible-playbook playbooks/demo/lb_pool_preview.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e azure_resource_group=my-rg \
  --vault-id @prompt
```

## Ad hoc examples

```bash
# AGW disconnect — the Application Gateway/backend pool are discovered from
# the target VM's NIC (see docs/setup.md#application-gateway-backend-discovery);
# no agw_name/agw_backend_pool_name to pass here. target_vm_ip is likewise
# resolved automatically from the same NIC lookup.
ansible-playbook playbooks/demo/lb_drain_disconnect.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e azure_resource_group=my-rg

# AGW reconnect — restores the association saved by the disconnect run above
ansible-playbook playbooks/demo/lb_reconnect.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e azure_resource_group=my-rg

# F5 reconnect — target_vm_ip is resolved automatically from
# target_vm_hostname (tasks/resolve_vm_private_ip.yml); pass -e
# target_vm_ip=<explicit IP> instead to override auto-discovery. Pool,
# partition, and member port are likewise discovered automatically from
# the resolved IP (see docs/setup.md#f5-pool-and-partition-discovery) — no
# f5_pool_name needed. Pass -e f5_pool_name=app_pool instead to bypass
# discovery and restrict the action to one specific pool.
ansible-playbook playbooks/demo/lb_reconnect.yml \
  -e lb_backend_type=f5 \
  -e target_vm_hostname=azure-vm-f501 \
  -e azure_resource_group=my-rg \
  -e f5_server=10.0.2.5 \
  -e f5_username=admin \
  -e f5_password=secret
```

## Teardown

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```
