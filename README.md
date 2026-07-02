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

## Usage

Run the playbook from the repository root:

```bash
ansible-playbook -i inventory/hosts.yml site.yml -e @extra-vars.yml
```

The role will:
1. Resolve the target VM on the specified Proxmox node.
2. Delete the VM from Proxmox using qm destroy.
3. Search NetBox for IP address records matching the VM name and delete them when cleanup is enabled.
