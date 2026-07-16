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

- `[ALWAYS REQUIRED]` — needed regardless of mode: AAP connection, object names, Git repo, credential names, the Azure connection and AGW/VM target identity variables (`azure_subscription_id`, `azure_tenant_id`, `azure_resource_group`, `agw_vm_hostname`, `agw_vm_private_ip`, `agw_name`, `agw_backend_pool_name`, `f5_vm_hostname`, `f5_vm_private_ip`), and the F5 BIG-IP connection variables (`f5_server`, `f5_username`, `f5_pool_name`, `f5_pool_member_port`, `f5_partition`) — the scenario job templates authenticate against and target these objects in both modes. `vault_azure_client_id`/`vault_azure_client_secret` are further conditional on a second, independent axis: `azure_auth_mode` (see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity)) — only required when it is `service_principal` (the default), unused when it is `msi`.
- `[LAB/DEV ONLY]` — consumed exclusively by `Setup - Azure infrastructure` / `Teardown - Azure infrastructure`: `azure_create_network_resources` and the whole create-from-scratch networking/VM block (`azure_vnet_name`, `azure_subnet_name`, `azure_nsg_name`, address prefixes, AGW SKU settings, `azure_vm_size`, image settings, `azure_vm_admin_username`, `azure_vm_ssh_public_key`) in `demo_variables.yml.example`. No secret in this demo is exclusively lab/dev-only today — see `vault.yml.example`.

Both `playbooks/aap_config.yml` (pre-task assertions) and `playbooks/verify.yml` enforce this split: the `[LAB/DEV ONLY]` checks only run `when: demo_manage_infrastructure | bool` (and, for `azure_vm_ssh_public_key`, additionally `when: azure_create_network_resources | bool`), so a customer/PoC deployment fails fast only on the variables it actually needs, never on unrelated provisioning variables.

When `demo_manage_infrastructure: false`:

- Set `azure_resource_group` / `agw_name` / `agw_backend_pool_name` / `agw_vm_private_ip` / `f5_server` / `f5_pool_name` / `f5_vm_private_ip` to the customer's existing resources.
- Request Azure/F5 credentials scoped to **read + pool-membership-write only** — Contributor-level Azure permissions and F5 LTM pool-member write access are not needed for creating VNets/VMs/AGW, since no provisioning/teardown job template exists to use them.
- Switching modes later: change the flag and re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt`. Going from `true` to `false` does **not** remove the infra job/workflow templates already created in AAP (dispatch only reconciles objects it is told about); delete them explicitly with `ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt` and re-apply, or delete them manually from the Controller UI.

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
| Azure subscription | Service Principal (default) or Managed Identity with AGW write permissions — see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity) |
| F5 BIG-IP | Existing LTM instance with pool (BYO — not provisioned by this demo) |

## Supporting dependencies

Every dependency listed below must exist before the demo workflows can run.
The setup playbooks in `playbooks/setup/` automate all Azure resources.
F5 BIG-IP is bring-your-own (BYO); the F5 setup playbook registers pool members only.

| Dependency | Category | Playbook | Job template |
|---|---|---|---|
| Azure VNet, subnet, NSG | Cloud networking | `playbooks/setup/01_azure_setup.yml` (create-from-scratch mode) | Setup - Azure infrastructure |
| Azure VMs (AGW backend + F5 member) | Cloud VMs | `playbooks/setup/01_azure_setup.yml` | Setup - Azure infrastructure |
| Azure AGW backend pool membership | Cloud-native | `playbooks/setup/01_azure_setup.yml` | Setup - Azure infrastructure |
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
Supply `azure_resource_group`, `agw_vm_private_ip`, `agw_name`, `agw_backend_pool_name`,
`f5_vm_private_ip`, `f5_server`, `f5_pool_name`.

**Create-from-scratch (`azure_create_network_resources: true`):**
Uncomment the create-from-scratch block in `demo_variables.yml` and supply naming variables.
The setup playbook will create the VNet, subnet, NSG, VMs, and AGW backend pool.

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
resource names.  Copy these into `demo_variables.yml` if using bring-your-own mode on a
subsequent run.

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

Parent inventory `Demo-Multicloud` with child `Azure-Resources` hosts both demo VMs.
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
