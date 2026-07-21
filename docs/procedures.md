# Demo procedures

## Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml -i inventory.yml --vault-id @prompt
```

## Workflow 1 — Disconnect VM

1. Launch **WF - Disconnect VM from load balancer**.
2. Survey: `lb_backend_type` = `agw` or `f5`, target hostname and IP.
3. Observe nodes: connectivity check → pool status preview → verify presence → verify status → drain/disconnect → collect results.

## Workflow 2 — Reconnect VM

1. Launch **WF - Reconnect VM to load balancer**.
2. Provide the same survey values used for disconnect.
3. Observe nodes: connectivity check → pool status preview → reconnect → verify status → collect results.

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
ansible-playbook playbooks/demo/lb_pool_preview.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e target_vm_ip=10.0.1.10 \
  -e azure_resource_group=my-rg \
  --vault-id @prompt
```

## Ad hoc examples

```bash
# AGW disconnect — the Application Gateway/backend pool are discovered from
# the target VM's NIC (see docs/setup.md#application-gateway-backend-discovery);
# no agw_name/agw_backend_pool_name to pass here.
ansible-playbook playbooks/demo/lb_drain_disconnect.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e target_vm_ip=10.0.1.10 \
  -e azure_resource_group=my-rg

# AGW reconnect — restores the association saved by the disconnect run above
ansible-playbook playbooks/demo/lb_reconnect.yml \
  -e lb_backend_type=agw \
  -e target_vm_hostname=azure-vm-agw01 \
  -e target_vm_ip=10.0.1.10 \
  -e azure_resource_group=my-rg

# F5 reconnect
ansible-playbook playbooks/demo/lb_reconnect.yml \
  -e lb_backend_type=f5 \
  -e target_vm_ip=10.0.1.11 \
  -e target_vm_hostname=azure-vm-f501 \
  -e f5_server=10.0.2.5 \
  -e f5_username=admin \
  -e f5_password=secret \
  -e f5_pool_name=app_pool
```

## Teardown

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```
