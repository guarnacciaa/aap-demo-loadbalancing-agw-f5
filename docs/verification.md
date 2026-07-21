# Verification

## Smoke test

```bash
ansible-playbook playbooks/verify.yml
```

## Optional pre-flight dry-run checks

Read-only; never create, modify, or delete any resource. Run before either scenario
workflow to confirm Azure/F5 credentials and current pool state — or rely on the
connectivity check and pool preview nodes that already run automatically as the first
two nodes of both workflows (see [docs/procedures.md](procedures.md)).

```bash
ansible-playbook playbooks/demo/lb_precheck_connectivity.yml \
  -e target_vm_hostname=azure-vm-agw01 -e azure_resource_group=my-rg --vault-id @prompt
ansible-playbook playbooks/demo/lb_pool_preview.yml \
  -e lb_backend_type=agw -e target_vm_hostname=azure-vm-agw01 \
  -e target_vm_ip=10.0.1.10 -e azure_resource_group=my-rg --vault-id @prompt
```

## Functional checks

### AGW

AGW backend pool membership is NIC-based in this environment (see
[docs/setup.md#application-gateway-backend-discovery](setup.md#application-gateway-backend-discovery)),
so verification is against the VM's NIC, not a static backend IP address:

- Portal: on the Application Gateway's backend pool **Targets** tab, the VM's network
  interface is no longer listed after the disconnect workflow.
- Portal: the VM's own **Networking** blade no longer shows the Application Gateway
  backend pool association on its NIC after disconnect.
- After the reconnect workflow, the NIC reappears in the pool's Targets tab.
- The VM's `lb_agw_backend_pool_id` tag holds the removed pool's resource ID between the
  disconnect and reconnect runs — inspect it via `az vm show -g <rg> -n <vm> --query tags`
  to confirm state was saved/restored correctly.

### F5

```bash
# TMSH (on BIG-IP) — member should show disabled/offline after disconnect
tmsh show ltm pool <pool> members
```

## References

- [azure_rm_networkinterface](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_networkinterface_module.html) — manages NIC-based AGW backend pool membership
- [azure_rm_virtualmachine_info](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_virtualmachine_info_module.html) — resolves the target VM's NIC for discovery
- [azure_rm_appgateway](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_appgateway_module.html) — read-only lookups only; this demo never writes to the Application Gateway resource
- [f5networks.f5_modules.bigip_pool_member](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/bigip_pool_member_module.html)
