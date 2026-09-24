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
  node: pve01.lab.example.com

netbox_cleanup: true
netbox:
  api_url: "https://netbox.example.local"
  token: "YOUR_NETBOX_TOKEN"
  ssl_verify: true

technitium_dns:
  enabled: true
  api_port: 53443
  validate_certs: false
  zone: "lab.example.com"

vms_to_delete:
  - name: W25C-TEST001
    vmid: 3021
    force: true
```

The values above (`example.com`, `pve01`) are placeholders. Supply your real environment through
AWX or a gitignored `extra-vars.yml`. Never commit tokens.

## Usage

Run the playbook from the repository root:

```bash
ansible-playbook -i inventory/hosts.yml site.yml -e @extra-vars.yml
```

The role will:

1. Resolve the target VM on the specified Proxmox node.
2. Query the VM's configured IPv4 address and remove matching Technitium PTR and A records when enabled.
3. Delete the VM from Proxmox using `qm destroy --purge`. This always deletes the VM's disks.
4. Remove the VM's `user-data-<vmid>` / `cloudbase-<vmid>` snippets left by proxmox-deploy.
5. When cleanup is enabled, search NetBox for the VM and delete only the IP address records that
   match exactly: same address as the VM's `ipconfig0`, or a `dns_name` whose host part is the VM
   name.

Technitium cleanup derives the reverse lookup name from the VM IP and queries that
name for PTR records, so no reverse zone needs to be configured. The forward zone
may be overridden per VM with `dns_zone`, and the full record name with `dns_name`.

For AWX, inject the credential as `technitium_dns_api_url` and
`technitium_dns_api_token`. Keep the API token out of job extra vars.
