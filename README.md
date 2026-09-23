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
  - tasks/technitium_dns.yml

## Example AWX extra vars

```yaml
proxmox:
  node: pve01.lab.sal9000.tech

netbox_cleanup: true
netbox:
  api_url: "https://netbox.example.local"
  token: "YOUR_NETBOX_TOKEN"
  ssl_verify: true

technitium_dns:
  enabled: true
  api_url: "https://dns.example.com"
  api_port: 53443
  api_token: "YOUR_API_TOKEN"
  validate_certs: true
  zone: "lab.sal9000.tech"

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
2. Query the VM's configured IPv4 address and remove matching Technitium PTR and A records when enabled.
3. Delete the VM from Proxmox using qm destroy.
4. Search NetBox for IP address records matching the VM name and delete them when cleanup is enabled.

Technitium cleanup derives the reverse lookup name from the VM IP and queries that
name for PTR records, so no reverse zone needs to be configured. The forward zone
may be overridden per VM with `dns_zone`, and the full record name with `dns_name`.
