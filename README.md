# proxmox-deprovision

This repository deprovisions VMs from Proxmox and can remove matching NetBox IP address records.

## Role layout

- role: proxmox_deprovision
  - defaults/main.yml
  - tasks/main.yml
  - tasks/deprovision_vm.yml
  - tasks/resolve_vars.yml
  - tasks/proxmox.yml
  - tasks/netbox.yml

## Example AWX extra vars

```yaml
proxmox:
  node: pve01.lab.sal9000.tech
  cluster_nodes:
    - pve01.lab.sal9000.tech
    - pve02.lab.sal9000.tech
    - pve03.lab.sal9000.tech

netbox_cleanup: true
netbox:
  api_url: "https://netbox.example.local"
  token: "YOUR_NETBOX_TOKEN"
  ssl_verify: true

vms_to_delete:
  - name: W25C-TEST001
    vmid: 3021
    force: true
    destroy_disk: true
```

## AWX / inventory guidance

This role can be driven either from AWX inventory variables or from job extra vars.

- Best fit for AWX: job extra vars or inventory group vars for `vms_to_delete` and `proxmox.cluster_nodes`.
  This keeps the VM list and cluster-node list explicit for each deprovision run.
- The Proxmox inventory host can still provide the base connection details, such as the primary node and SSH credentials, but it is usually not the best place to store the per-run VM list.
- If you want the VM list to come from inventory, you can also place `vms_to_delete` under a group or host var, but that is less flexible for one-off deletions than extra vars.

## Usage

Run the playbook from the repository root:

```bash
ansible-playbook -i inventory/hosts.yml site.yml -e @extra-vars.yml
```

The role will:
1. Resolve the target VM on the specified Proxmox node.
2. Delete the VM from Proxmox using qm destroy.
3. Search NetBox for IP address records matching the VM name and delete them when cleanup is enabled.
