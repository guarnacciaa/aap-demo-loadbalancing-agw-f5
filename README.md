# AGW and F5 Load Balancer VM Management Demo

![Status](https://img.shields.io/badge/Status-WIP-yellow)
![Red Hat Ansible Automation Platform](https://img.shields.io/badge/AAP-2.6-red)
![Configuration as Code](https://img.shields.io/badge/CasC-infra.aap_configuration-blue)
![Azure AGW](https://img.shields.io/badge/Azure-Application_Gateway-0078d4)
![F5 BIG-IP](https://img.shields.io/badge/F5-f5__modules-e4002b)

## Introduction

Automates **verification, drain, disconnection, and reconnection** of VMs behind **Azure Application Gateway** and **F5 BIG-IP** (Use Case 1). Two Azure VMs demonstrate AGW and F5 paths through AAP workflow templates.

## How to run the demo

| Phase | Workflow / template | Purpose | Mode |
|---|---|---|---|
| 0. Setup (optional) | `WF - Demo setup` | Provision Azure infrastructure and register the F5 pool member | Lab/dev only |
| 1. Scenario | `WF - Disconnect VM from load balancer` | Connectivity check → pool preview → verify presence → verify status → drain/disconnect → collect | Always |
| 1. Scenario | `WF - Reconnect VM to load balancer` | Connectivity check → pool preview → reconnect → verify status → collect | Always |
| 2. Teardown (optional) | `WF - Demo teardown` | Remove F5 pool membership and deprovision Azure resources | Lab/dev only |

Phases 0 and 2 only exist when `demo_manage_infrastructure: true` (default); see [Deployment modes](#deployment-modes) for the customer/PoC mode that runs phase 1 only, against pre-existing infrastructure. Both scenario workflows now start with a read-only connectivity check and pool preview (see [Job templates](#job-templates)) before making any change.

Launch either scenario workflow from AAP **Templates** with survey values for `lb_backend_type`, hostname, and IP.

## Application Gateway backend discovery

This environment associates AGW backend pool membership with the target VM's **NIC ipConfiguration**, not with a static backend IP address. Every `LB - *` scenario and dry-run template therefore discovers the Application Gateway (name, resource group, backend pool) dynamically at runtime from `agw_vm_hostname`'s primary network interface — see `playbooks/demo/tasks/agw_discover_from_nic.yml`. There is no `agw_name` / `agw_backend_pool_name` variable to configure for these templates; only `agw_vm_hostname` and `azure_resource_group` (the VM's resource group) are needed.

`LB - Drain and disconnect VM` records the removed backend pool's resource ID in a `lb_agw_backend_pool_id` tag on the VM, so `LB - Reconnect VM to pool` — potentially launched much later, in a separate job run — can restore the exact same NIC-to-pool association without it being passed in again (mirroring how a stateful detach/reattach controller would persist this information). `LB - Pool status preview (dry run)` reports both the live association and, when absent, this saved one.

`agw_name` / `agw_backend_pool_name` still exist as variables, but only `Setup - Azure infrastructure` reads them (lab/dev only), to perform the *initial* NIC attachment when the demo bootstraps its own environment — see [Deployment modes](#deployment-modes) and [docs/setup.md](docs/setup.md#application-gateway-backend-discovery).

**Limitation:** only the VM's first network interface and its primary IP configuration are inspected — multi-NIC VMs, or a pool attached to a secondary NIC/IP configuration, are out of scope for this demo.

## Azure authentication mode

`azure_auth_mode` in `demo_variables.yml` (default `service_principal`) selects how every `azure.azcollection` task authenticates, independent of the deployment mode below:

| `azure_auth_mode` | Requires | Notes |
|---|---|---|
| `service_principal` (default) | `vault_azure_client_id` / `vault_azure_client_secret` | Credentials injected by AAP's "Azure Service Principal" credential. Works on any AAP topology and is the only option if AAP does not run on Azure. |
| `msi` | Nothing stored in AAP | `auth_source: msi` on every `azure_rm_*` task acquires a token from the Azure Instance Metadata Service (IMDS) endpoint. AAP's execution node/EE container must itself be an Azure resource with a Managed Identity enabled — see [docs/setup.md](docs/setup.md#azure-authentication-mode-service-principal-vs-managed-identity). |

`azure_auth_mode` must be threaded through `extra_vars` on every Azure-facing job template (already done in `group_vars/all/job_templates.yml` / `job_templates_infra.yml`): `playbooks/demo/*.yml` and `playbooks/setup/*.yml` run against AAP's generated inventory, which does not reliably auto-load `group_vars/all/demo_variables.yml`.

## Deployment modes

`demo_manage_infrastructure` in `demo_variables.yml` (default `true`) controls how much of the AAP catalog this CasC creates:

| Mode | `demo_manage_infrastructure` | AAP objects created | Use when |
|---|---|---|---|
| Lab / dev | `true` (default) | Full lifecycle: `Setup - Azure infrastructure`, `Setup - F5 pool member`, `Teardown - Azure infrastructure`, `Teardown - F5 pool member`, `WF - Demo setup`, `WF - Demo teardown`, plus all scenario and dry-run objects | You are running the demo yourself and want AAP to create and destroy the Azure VMs/AGW and register the F5 pool member |
| Customer / PoC | `false` | Only the scenario job templates (`LB - Connectivity check (dry run)`, `LB - Pool status preview (dry run)`, `LB - Verify VM in pool`, `LB - Verify connection status`, `LB - Drain and disconnect VM`, `LB - Reconnect VM to pool`, `LB - Collect results`) and their two workflows | The customer already provides the Azure VM/AGW and F5 pool member; no provisioning or teardown object is created in AAP, removing any risk of accidentally launching a job that creates duplicate Azure resources or removes the customer's existing F5 pool member |

In customer mode, set `azure_resource_group` / `agw_vm_hostname` / `agw_vm_private_ip` / `f5_server` / `f5_pool_name` / `f5_vm_private_ip` to the customer's existing resources. `agw_name` / `agw_backend_pool_name` do **not** need to be set in customer mode — the Application Gateway is discovered dynamically from the VM's NIC (see [Application Gateway backend discovery](#application-gateway-backend-discovery)); those two variables only matter to `Setup - Azure infrastructure`, which is not even deployed in customer mode.

`group_vars/all/demo_variables.yml.example` **and** `vault.yml.example` mark every variable/secret with `[ALWAYS REQUIRED]` or `[LAB/DEV ONLY]` banners so you can see at a glance what customer/PoC mode needs. `playbooks/aap_config.yml` and `playbooks/verify.yml` enforce this: the `[LAB/DEV ONLY]` variables are only validated when `demo_manage_infrastructure: true`, so leaving them at their example defaults never blocks a customer/PoC deployment.

## Quick start

```bash
cd artifacts/demos/aap-demo-loadbalancing-agw-f5
ansible-galaxy collection install -r collections/requirements.yml -p collections
cp ansible.cfg.example ansible.cfg
cp group_vars/all/demo_variables.yml.example group_vars/all/demo_variables.yml
cp vault.yml.example vault.yml && ansible-vault encrypt vault.yml
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

See [docs/setup.md](docs/setup.md) for prerequisites, EE build instructions, and the setup workflow.

## Reset / teardown

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```

## Architecture

```mermaid
flowchart TB
  subgraph aap [AAP]
    WFD[WF_Disconnect]
    WFR[WF_Reconnect]
  end

  subgraph azure [Azure]
    AGW[Application_Gateway]
    VM1[VM_AGW_backend]
  end

  subgraph f5 [F5_BIG_IP]
    Pool[LTM_pool]
    VM2[VM_F5_member]
  end

  WFD --> AGW
  WFD --> Pool
  VM1 --> AGW
  VM2 --> Pool
  WFR --> AGW
  WFR --> Pool
```

## Multicloud inventory UX (UC4)

```
Demo-LoadBalancingAGWF5
└── Azure-Resources-LoadBalancingAGWF5
    ├── azure_vms (AGW + F5 VMs)
    └── azure_loadbalancers (AGW/F5 metadata)
```

## Job templates

The **Setup** and **Teardown** templates are only created in AAP when `demo_manage_infrastructure: true` (see [Deployment modes](#deployment-modes)); the **scenario** and **dry-run** templates are always created and are wired into both scenario workflows (see [How to run the demo](#how-to-run-the-demo)).

| Template | Playbook | Mode |
|---|---|---|
| LB - Verify VM in pool | `demo/lb_verify_presence.yml` | Always |
| LB - Verify connection status | `demo/lb_verify_status.yml` | Always |
| LB - Drain and disconnect VM | `demo/lb_drain_disconnect.yml` | Always |
| LB - Reconnect VM to pool | `demo/lb_reconnect.yml` | Always |
| LB - Collect results | `demo/lb_collect_results.yml` | Always |
| Setup - Azure infrastructure | `setup/01_azure_setup.yml` | Lab/dev only |
| Setup - F5 pool member | `setup/02_f5_setup.yml` | Lab/dev only |
| Teardown - Azure infrastructure | `setup/01_azure_teardown.yml` | Lab/dev only |
| Teardown - F5 pool member | `setup/02_f5_teardown.yml` | Lab/dev only |

### Dry-run / preview templates

Read-only checks, useful in both deployment modes. Wired as the first two nodes of both scenario workflows (ahead of any mutating task) and also launchable standalone.

| Template | Playbook | Checks |
|---|---|---|
| LB - Connectivity check (dry run) | `demo/lb_precheck_connectivity.yml` | Confirms Azure (via the target VM/NIC) and the F5 BIG-IP API are both reachable with the configured credentials — never mutates anything |
| LB - Pool status preview (dry run) | `demo/lb_pool_preview.yml` | Reports whether the target VM is currently present in the AGW pool (discovered from its NIC) or as an F5 member, i.e. what drain/disconnect or reconnect would actually change — never mutates anything |

## Collections

| Collection | Tier | Purpose |
|---|---|---|
| infra.aap_configuration | validated | CasC |
| azure.azcollection | certified | Application Gateway |
| f5networks.f5_modules | certified | F5 pool members |
| ansible.netcommon | certified | F5 httpapi connection |
| ansible.controller | certified | Post-CasC EE/inventory wiring in `aap_config.yml`; explicit object deletion in `aap_cleanup.yml` |

## References

- Red Hat AAP 2.6 documentation
- Azure Application Gateway product documentation
- F5 BIG-IP iControl REST API documentation
