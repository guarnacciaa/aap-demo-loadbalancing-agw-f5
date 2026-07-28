# Environment setup

## Deployment mode: lab/dev vs customer/PoC

Set `demo_manage_infrastructure` in `group_vars/all/demo_variables.yml` **before** running `aap_config.yml`:

| `demo_manage_infrastructure` | Scenario | What CasC creates |
|---|---|---|
| `true` (default) | Lab, dev, or self-running the full demo; no pre-existing Azure VM/AGW or F5 pool member | `Setup - Azure infrastructure`, `Setup - F5 pool member`, `Teardown - Azure infrastructure`, `Teardown - F5 pool member` job templates, `WF - Demo setup`, `WF - Demo teardown`, plus all scenario and dry-run objects |
| `false` | Deploying at a customer site where the Azure VM/AGW and F5 pool member already exist | Only the scenario job templates (`LB - Connectivity check (dry run)`, `LB - Pool status preview (dry run)`, `LB - Verify VM in pool`, `LB - Verify connection status`, `LB - Drain and disconnect VM`, `LB - Reconnect VM to pool`, `LB - Collect results`) and their two workflows |

The flag is read in `playbooks/aap_config.yml`: when `true`, the objects defined in `group_vars/all/job_templates_infra.yml` and `group_vars/all/workflow_templates_infra.yml` are merged into the lists the `infra.aap_configuration` dispatch role applies; when `false`, those two files are still loaded (Ansible auto-loads every file under `group_vars/all/`) but never merged in, so their objects are never created in AAP.

### Which variables to set in each mode

`group_vars/all/demo_variables.yml.example` **and** `vault.yml.example` group every variable/secret under the same banner:

- `[ALWAYS REQUIRED]` — needed regardless of mode: AAP connection, object names, Git repo, credential names, the Azure connection and AGW/VM target identity variables (`azure_subscription_id`, `azure_tenant_id`, `azure_resource_group`, `agw_vm_hostname`, `f5_vm_hostname`), and the F5 BIG-IP connection variables (`f5_server`, `f5_username`) — the scenario job templates authenticate against and target these objects in both modes. `vault_azure_client_id`/`vault_azure_client_secret` are further conditional on a second, independent axis: `azure_auth_mode` (see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity)) — only required when it is `service_principal` (the default), unused when it is `msi`. `agw_vm_private_ip` / `f5_vm_private_ip` are **optional overrides**, not required in either mode — see [Automatic private IP discovery](#automatic-private-ip-discovery).
- `[LAB/DEV ONLY]` — consumed exclusively by `Setup - Azure infrastructure` / `Teardown - Azure infrastructure`: `agw_name`, `agw_backend_pool_name` (see [Application Gateway backend discovery](#application-gateway-backend-discovery) below), `azure_create_network_resources`, and the whole create-from-scratch networking/VM block (`azure_vnet_name`, `azure_subnet_name`, `azure_nsg_name`, address prefixes, AGW SKU settings, `azure_vm_size`, image settings, `azure_vm_admin_username`, `azure_vm_ssh_public_key`); and by `Setup - F5 pool member` / `Teardown - F5 pool member`: `f5_pool_name`, `f5_partition` (see [F5 pool and partition discovery](#f5-pool-and-partition-discovery) below) — in `demo_variables.yml.example`. `f5_pool_member_port` is a further **optional override** within that same lab/dev scope, defaulting to `80` when left unset. No secret in this demo is exclusively lab/dev-only today — see `vault.yml.example`.

Both `playbooks/aap_config.yml` (pre-task assertions) and `playbooks/verify.yml` enforce this split: the `[LAB/DEV ONLY]` checks only run `when: demo_manage_infrastructure | bool` (and, for `azure_vm_ssh_public_key`, additionally `when: azure_create_network_resources | bool`), so a customer/PoC deployment fails fast only on the variables it actually needs, never on unrelated provisioning variables.

When `demo_manage_infrastructure: false`:

- Set `azure_resource_group` / `agw_vm_hostname` / `f5_server` / `f5_vm_hostname` to the customer's existing resources. Do **not** set `agw_name` / `agw_backend_pool_name` / `f5_pool_name` / `f5_partition` — they are lab/dev-only (see [Application Gateway backend discovery](#application-gateway-backend-discovery) and [F5 pool and partition discovery](#f5-pool-and-partition-discovery)). `agw_vm_private_ip` / `f5_vm_private_ip` / `f5_pool_member_port` do not need to be set either — see [Automatic private IP discovery](#automatic-private-ip-discovery) (the first two) and the note above (the third, defaults to `80`).
- Request Azure/F5 credentials scoped to **read + NIC write + pool-membership-write only** — Contributor-level Azure permissions and F5 LTM pool-member write access are not needed for creating VNets/VMs/AGW, since no provisioning/teardown job template exists to use them. The Azure role must additionally cover `Microsoft.Network/networkInterfaces/write` (NIC-based backend pool attach/detach) and `Microsoft.Compute/virtualMachines/write` (the `lb_agw_backend_pool_id` state tag written by `LB - Drain and disconnect VM` / `LB - Reconnect VM to pool`) — the built-in **Network Contributor** role on the VM's resource group covers both.
- Switching modes later: change the flag and re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt`. Going from `true` to `false` does **not** remove the infra job/workflow templates already created in AAP (dispatch only reconciles objects it is told about); delete them explicitly with `ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt` and re-apply, or delete them manually from the Controller UI.

## Application Gateway backend discovery

This environment associates AGW backend pool membership with the target VM's **NIC ipConfiguration**, not with a static backend IP address. Every `LB - *` scenario and dry-run template discovers the Application Gateway — name, resource group, and backend pool — dynamically at runtime from `agw_vm_hostname`'s primary network interface, instead of reading it from a fixed `agw_name`/`agw_backend_pool_name` variable. See `playbooks/demo/tasks/agw_discover_from_nic.yml` for the implementation.

What this means in practice:

- Only `agw_vm_hostname` and `azure_resource_group` (the VM's resource group) are needed to identify the Application Gateway — in **every** deployment mode, including customer/PoC.
- `agw_name` / `agw_backend_pool_name` are `[LAB/DEV ONLY]`: the only consumer is `Setup - Azure infrastructure`, which performs the *initial* NIC attachment when the demo bootstraps its own environment (both the bring-your-own and create-from-scratch sub-modes). They are never read by any `LB - *` scenario template.
- `LB - Drain and disconnect VM` saves the removed backend pool's resource ID in a `lb_agw_backend_pool_id` tag on the VM. `LB - Reconnect VM to pool` reads that tag to restore the exact same association — this is what makes reconnect work correctly even when launched as a separate job run, potentially much later, once the NIC no longer carries a live association to discover. `LB - Pool status preview (dry run)` reports this saved state too, when the VM currently has no live association.
- Required RBAC changed accordingly: the Azure credential needs NIC write and VM tag write (`Network Contributor` on the VM's resource group), not Application Gateway write — this demo never modifies the Application Gateway resource itself, only NIC-side backend pool membership.
- **Limitation**: only the VM's first network interface and its primary IP configuration are inspected. Multi-NIC VMs, or a pool attached to a secondary NIC or non-primary IP configuration, are out of scope for this demo.

## F5 pool and partition discovery

F5 BIG-IP ties pool membership to a **pool + partition + port** combination (`f5networks.f5_modules.bigip_pool_member`), and the same node IP can be a member of several pools at once — in the same or different partitions, at the same or different ports. Every `LB - *` scenario and dry-run template discovers **every** such membership dynamically at runtime from the target VM's resolved private IP, instead of reading a fixed `f5_pool_name`/`f5_partition`/`f5_pool_member_port` variable. See `playbooks/demo/tasks/f5_discover_pool_from_ip.yml` for the implementation.

What this means in practice:

- Only `f5_server`/`f5_username` (with `f5_password` from vault) are needed to authenticate — in **every** deployment mode, including customer/PoC. No pool, partition, or port needs to be known in advance.
- **Management port auto-detection**: neither the raw `uri` calls (`tasks/f5_authenticate.yml`, `tasks/f5_discover_pool_from_ip.yml`, `demo/lb_precheck_connectivity.yml`) nor `f5networks.f5_modules.bigip_pool_member`'s `provider.server_port` hard-code a port. `tasks/f5_resolve_management_port.yml` probes TCP reachability on `443` first, falling back to `8443` (the port BIG-IP VE marketplace/lab images commonly use for management REST, since they reserve `443` for the data-plane virtual server) — the resolved port is cached as `f5_resolved_management_port` for the rest of the play. Set `f5_management_port` explicitly in `demo_variables.yml` to skip probing when the target F5 uses neither port.
- **Authentication**: `tasks/f5_discover_pool_from_ip.yml` and `demo/lb_precheck_connectivity.yml` authenticate via `tasks/f5_authenticate.yml`, a token-based login (`POST /mgmt/shared/authn/login` with `{username, password, loginProviderName}`, then `X-F5-Auth-Token` on every subsequent call) instead of HTTP Basic Auth. Basic Auth against `/mgmt/tm/...` does not authenticate on every BIG-IP; some environments require this token endpoint instead. `f5_login_provider_name` (default `tmos`) must match the configured auth partition name for `f5_username` — see `demo_variables.yml.example`.
- **`demo_no_log` must be threaded through extra_vars on every job template whose playbook can reach `tasks/f5_authenticate.yml`** (see `group_vars/all/job_templates.yml` / `job_templates_infra.yml`), for the exact same reason documented for `azure_auth_mode` below: `group_vars/all/demo_variables.yml` is not auto-loaded for `playbooks/demo/*.yml` / `playbooks/setup/*.yml` under AAP's generated inventory. Without it, `no_log: "{{ demo_no_log | default(true) }}"` in `f5_authenticate.yml` always falls back to `true`, censoring the F5 login task's output even when `demo_no_log: false` is set in `demo_variables.yml` — this is what previously blocked diagnosing the F5 authentication failures logged in `docs/testing.md`.
- Discovery issues a single `GET /mgmt/tm/ltm/pool/?expandSubcollections=true` call, which returns every pool (with members expanded inline) visible to the F5 API credential, then matches every member whose address equals the target IP — regardless of which pool or partition it is in.
- `LB - Drain and disconnect VM` / `LB - Reconnect VM to pool` loop over **every** discovered membership: draining or reconnecting a node touches all of its pool memberships at once, not just one. `bigip_pool_member` with `state: forced_offline`/`disabled` never removes the member record itself, so a disabled membership remains discoverable — this is what makes reconnect find it again later, in a separate job run.
- `f5_pool_name`/`f5_partition`/`f5_pool_member_port` are `[LAB/DEV ONLY]`: the only consumers are `Setup - F5 pool member` / `Teardown - F5 pool member`, which need an explicit pool identity to create (or remove) the *very first* membership — before that membership exists, there is nothing for discovery to find. They are never read by any `LB - *` scenario template, though `f5_pool_name` can still be passed as an explicit override (`-e f5_pool_name=...`) at launch to bypass discovery and restrict an action to one specific pool. `f5_pool_member_port` specifically is a further **optional override** on top of that: both `Setup`/`Teardown - F5 pool member` fall back to `80` (`| default(80)` in the tasks themselves) when it is left commented out in `demo_variables.yml.example` — set it explicitly only if the very first member must be registered on a different port (for example an HTTPS-only backend on `443`).
- Required RBAC: the F5 API credential needs read access to **every partition** the target node's pools might live in, plus pool-member write access to modify them. A credential scoped to a single partition limits discovery (and therefore drain/reconnect) to what it can see in that partition.
- **Limitations**: the member port is parsed from the member's `<address>:<port>` name field by splitting on the last `:`, which assumes IPv4 addresses (matching the IPv4-only scope of the rest of this demo). Discovery only sees partitions the F5 API credential can access.

## Automatic private IP discovery

`agw_vm_private_ip` and `f5_vm_private_ip` are **optional overrides**, not required variables, in every deployment mode. When left unset (the `.example` default — commented out), every consumer resolves the VM's current primary private IP automatically from its hostname:

| Consumer | How it resolves the IP |
|---|---|
| `LB - *` scenario/dry-run templates (AGW branch) | Reused at no extra cost from the NIC lookup `tasks/agw_discover_from_nic.yml` already performs for Application Gateway backend pool discovery |
| `LB - *` scenario/dry-run templates (F5 branch) | `tasks/resolve_vm_private_ip.yml` — the same VM/NIC lookup pattern, targeting `target_vm_hostname` |
| `Setup - Azure infrastructure` | Always looks up `agw_vm_hostname`/`f5_vm_hostname`'s NIC (both modes) — the create-from-scratch VMs are looked up the same way once created |
| `Setup - F5 pool member` / `Teardown - F5 pool member` | `tasks/resolve_vm_private_ip.yml` against `f5_vm_hostname`, unless `f5_vm_private_ip` is explicitly set |

Set `agw_vm_private_ip` / `f5_vm_private_ip` explicitly only as an override — for example when a VM has multiple NICs and the wrong one is auto-detected, or the pool member address must differ from the VM's primary private IP for some other reason. `group_vars/all/hosts.yml`'s `ansible_host` field (informational only — no playbook connects to these Controller host records over SSH) falls back to the VM hostname string when no override is set, so leaving both variables unset never blocks `aap_config.yml`.

## Azure authentication mode: Service Principal vs Managed Identity

`azure_auth_mode` in `group_vars/all/demo_variables.yml` (default `service_principal`) controls how every `azure.azcollection.azure_rm_*` task (in `01_azure_setup.yml`, `01_azure_teardown.yml`, and the `demo/lb_*.yml` playbooks) authenticates to Azure. This is independent of `demo_manage_infrastructure` above — it applies in every deployment mode.

| `azure_auth_mode` | How it authenticates | Requirements |
|---|---|---|
| `service_principal` (default) | The "Azure Service Principal" AAP credential injects `client_id`/`client_secret`/`tenant` as environment variables that `azure.azcollection` reads automatically | `vault_azure_client_id` and `vault_azure_client_secret` in `vault.yml`. Works regardless of where AAP is hosted. |
| `msi` | Every `azure_rm_*` task passes `auth_source: msi`, which makes the module request a token from the Azure Instance Metadata Service (IMDS) endpoint instead; no client secret is stored in AAP at all | The AAP execution node or execution environment container that actually runs the job must itself be an Azure resource (VM, VMSS, AKS node, etc.) with a system- or user-assigned Managed Identity enabled, and that identity must hold the same RBAC role documented for Service Principal below (Contributor for create-from-scratch, or the reduced scope in the deployment-mode section above for bring-your-own/customer mode). |

Notes:

- AAP's built-in "Microsoft Azure Resource Manager" credential type has **no Managed Identity input field** — it only supports Service Principal or Active Directory user/password. The Azure credential is still created in `msi` mode (with only `azure_subscription_id` populated) so job templates keep a stable credential association, but `client`/`secret`/`tenant` are omitted entirely from `inputs` — no secret is ever stored (see `group_vars/all/credentials.yml`).
- Choose `msi` only when you know AAP's execution nodes run on Azure infrastructure with an identity attached. In every other topology (on-prem, another cloud, OpenShift not on Azure), `service_principal` is the only option that works.
- `playbooks/aap_config.yml` and `playbooks/verify.yml` validate `azure_auth_mode` and only require the Service Principal vault secrets when it is set to `service_principal` (the default).
- Switching modes: change `azure_auth_mode` and re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt` to update the Azure credential's stored inputs.
- **`azure_auth_mode` must be threaded through each job template's `extra_vars`** (see `group_vars/all/job_templates.yml` / `job_templates_infra.yml`). AAP runs `playbooks/demo/*.yml` and `playbooks/setup/*.yml` against its own generated inventory, not this repository's `inventory.yml`, so `group_vars/all/demo_variables.yml` is **not** auto-loaded for those nested playbooks the way it is for `playbooks/aap_config.yml` and `playbooks/verify.yml`. If a job template is missing `azure_auth_mode` in its `extra_vars`, the `auth_source` Jinja expression silently falls back to the `service_principal` branch — even with `azure_auth_mode: msi` set correctly in `demo_variables.yml`. Re-run `aap_config.yml` after editing `extra_vars` so the stored job template definitions pick up the change.

## Requirements

| Component | Version |
|---|---|
| Red Hat Ansible Automation Platform | 2.6+ |
| `ansible-builder` | 3.x (for EE build) |
| Podman | Latest stable |
| Azure subscription | Service Principal (default) or Managed Identity with NIC write + VM tag write permissions (`Network Contributor` on the VM's resource group) — see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity) and [Application Gateway backend discovery](#application-gateway-backend-discovery) |
| F5 BIG-IP | Existing LTM instance with pool (BYO — not provisioned by this demo) |

## Supporting dependencies

Every dependency listed below must exist before the demo workflows can run.
The setup playbooks in `playbooks/setup/` automate all Azure resources.
F5 BIG-IP is bring-your-own (BYO); the F5 setup playbook registers pool members only.

| Dependency | Category | Playbook | Job template |
|---|---|---|---|
| Azure VNet, subnet, NSG | Cloud networking | `playbooks/setup/01_azure_setup.yml` (create-from-scratch mode) | Setup - Azure infrastructure |
| Azure VMs (AGW backend + F5 member) | Cloud VMs | `playbooks/setup/01_azure_setup.yml` | Setup - Azure infrastructure |
| Azure AGW backend pool membership (initial NIC attach) | Cloud-native | `playbooks/setup/01_azure_setup.yml` | Setup - Azure infrastructure |
| F5 LTM pool member registration | Third-party service (BYO) | `playbooks/setup/02_f5_setup.yml` | Setup - F5 pool member |

## Step 1 — Install collections and prepare local config

```bash
cd artifacts/demos/aap-demo-loadbalancing-agw-f5
ansible-galaxy collection install -r collections/requirements.yml -p collections
cp ansible.cfg.example ansible.cfg
cp group_vars/all/demo_variables.yml.example group_vars/all/demo_variables.yml
cp vault.yml.example vault.yml && ansible-vault encrypt vault.yml
```

Edit `group_vars/all/demo_variables.yml` and fill in all `CHANGE_ME` placeholders.
Edit `vault.yml` (after decrypting) with real credentials.

## Step 2 — Configure Azure and F5 variables

In `demo_variables.yml`, choose your provisioning mode:

**Bring-your-own (default, `azure_create_network_resources: false`):**
Supply `azure_resource_group`, `agw_vm_hostname`, `agw_name`, `agw_backend_pool_name`,
`f5_vm_hostname`, `f5_server`, `f5_pool_name`, `f5_partition`. `agw_name`/`agw_backend_pool_name`
and `f5_pool_name`/`f5_partition` are only used by `Setup - Azure infrastructure` /
`Setup - F5 pool member` to create the *initial* NIC attachment / pool member on first
run — see [Application Gateway backend discovery](#application-gateway-backend-discovery)
and [F5 pool and partition discovery](#f5-pool-and-partition-discovery).
`agw_vm_private_ip`/`f5_vm_private_ip`/`f5_pool_member_port` do not need to be supplied — see
[Automatic private IP discovery](#automatic-private-ip-discovery) (the first two; `f5_pool_member_port`
defaults to `80`).

**Create-from-scratch (`azure_create_network_resources: true`):**
Uncomment the create-from-scratch block in `demo_variables.yml` and supply naming variables.
The setup playbook will create the VNet, subnet, NSG, and VMs, and attach the AGW VM's NIC
to the pre-existing Application Gateway named by `agw_name`/`agw_backend_pool_name` (this
demo never creates the Application Gateway resource itself, only backend pool membership).

## Step 3 — Build and push the custom Execution Environment

The demo requires `azure.azcollection` and `f5networks.f5_modules` at runtime.
Build and push `ee-loadbalancing-agw-f5` to Private Automation Hub before running CasC.

```bash
# Build the EE image
ansible-builder build \
  -f context/execution-environment.yml \
  -t <PAH-HOST>/ee-loadbalancing-agw-f5:latest \
  --container-runtime podman

# Push to PAH
podman login <PAH-HOST> --tls-verify=false
podman push <PAH-HOST>/ee-loadbalancing-agw-f5:latest --tls-verify=false
```

Set `demo_execution_environment_image: <PAH-HOST>/ee-loadbalancing-agw-f5:latest` in
`demo_variables.yml` before running CasC.

## Step 4 — Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

This registers the organization, credentials, project, inventories, EE, job templates,
and workflow templates in AAP.

## Step 5 — Run setup workflows from AAP

From **Templates** in the AAP UI, run in order:

1. **WF - Demo setup** — provisions Azure infrastructure (if create-from-scratch) and registers
   the F5 pool member.

The `set_stats` output from `Setup - Azure infrastructure` prints resolved VM IPs and
resource names, and (via AAP's workflow artifact chaining) hands `f5_vm_private_ip` to
`Setup - F5 pool member` automatically. No manual copy into `demo_variables.yml` is
needed — every subsequent run re-resolves the same values from Azure — see
[Automatic private IP discovery](#automatic-private-ip-discovery).

## Step 6 — Verify the environment

```bash
ansible-playbook playbooks/verify.yml --vault-id @prompt
```

Optionally, launch `LB - Connectivity check (dry run)` and `LB - Pool status preview
(dry run)` from the Controller UI, or run them locally — see
[docs/procedures.md](procedures.md) for the equivalent `ansible-playbook` invocations.
They never create, modify, or delete any resource, and also run automatically as the
first two nodes of both scenario workflows.

## Multicloud inventory

Parent inventory `Demo-LoadBalancingAGWF5` with child `Azure-Resources-LoadBalancingAGWF5` hosts both demo VMs.
Group `azure_loadbalancers` holds AGW/F5 metadata variables for templates.

## Teardown

To remove all demo objects from AAP:

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```

To deprovision Azure infrastructure (create-from-scratch mode only), run **WF - Demo teardown**
from the AAP UI before running `aap_cleanup.yml`.

## Vault usage

`vault.yml` contains:

| Variable | Purpose |
|---|---|
| `vault_controller_username` | AAP Controller username |
| `vault_controller_password` | AAP Controller password |
| `vault_azure_client_id` | Azure Service Principal client ID (only when `azure_auth_mode: service_principal`) |
| `vault_azure_client_secret` | Azure Service Principal client secret (only when `azure_auth_mode: service_principal`) |
| `vault_f5_password` | F5 BIG-IP API password |
