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
ansible-playbook playbooks/demo/lb_precheck_connectivity.yml --vault-id @prompt
ansible-playbook playbooks/demo/lb_pool_preview.yml \
  -e lb_backend_type=agw -e target_vm_ip=10.0.1.10 --vault-id @prompt
```

## Functional checks

### AGW

- Portal: backend pool no longer lists VM IP after disconnect workflow.
- After reconnect workflow, VM IP reappears in the pool.

### F5

```bash
# TMSH (on BIG-IP) — member should show disabled/offline after disconnect
tmsh show ltm pool <pool> members
```

## References

- [azure_rm_appgateway](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_appgateway_module.html)
- [f5networks.f5_modules.bigip_pool_member](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/bigip_pool_member_module.html)
