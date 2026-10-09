# proxmox-deprovision

This repository deprovisions VMs from Proxmox and cleans up after them: matching NetBox IP address records,
Technitium DNS records and, for Windows VMs, the Active Directory computer object.

## Role layout

- role: proxmox_deprovision
  - defaults/main.yml
  - tasks/main.yml
  - tasks/deprovision_vm.yml
  - tasks/resolve_vars.yml
  - tasks/proxmox.yml
  - tasks/netbox.yml
  - tasks/technitium_dns.yml
  - tasks/active_directory.yml

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

# Windows VMs: the inventory host that deletes AD computer objects (see "Active Directory cleanup")
ad_cleanup_host: mgmt-01.ad.example.com

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
5. For a Windows VM, delete its AD computer object (see below).
6. When cleanup is enabled, search NetBox for the VM and delete only the IP address records that
   match exactly: same address as the VM's `ipconfig0`, or a `dns_name` whose host part is the VM
   name.

Technitium cleanup derives the reverse lookup name from the VM IP and queries that
name for PTR records, so no reverse zone needs to be configured. The forward zone
may be overridden per VM with `dns_zone`, and the full record name with `dns_name`.

For AWX, inject the credential as `technitium_dns_api_url` and
`technitium_dns_api_token`. Keep the API token out of job extra vars.

## Active Directory cleanup

A VM is treated as Windows when its Proxmox OS type (`ostype`) is a Windows one (`win11` for Windows Server 2025;
they all start with `w`), the same rule proxmox-deploy uses; `os_type: windows` or `linux` on the entry overrides
it. After the VM is destroyed, its computer account is deleted with `microsoft.ad.computer` (`state: absent`),
child objects such as BitLocker recovery information included. A computer that's already gone is reported and
skipped, so re-runs are safe.

- **Which object:** `<NAME>$`, the VM's Proxmox name in capitals (proxmox-deploy names the computer after the
  VM). Set `ad_computer_name` on the entry when it differs, e.g. a VM deployed with its own `hostname`.
- **Where it runs:** on `ad_cleanup_host`, an inventory host with the ActiveDirectory PowerShell module: a
  domain controller, or a member server with `RSAT-AD-PowerShell`. The play connects to it only for this task,
  over PSRP with the connection settings it has in the inventory
  ([inventory/hosts.yml](inventory/hosts.yml) has an example in the `ad_management` group). With
  `ad_cleanup_host` unset, Windows VMs' objects are left in place and the job log says so.
- **Who deletes it:** `ad_cleanup_user` / `ad_cleanup_password`, by default the domain-join account
  (`domain_join_user` / `domain_join_pass`, the credential proxmox-deploy uses). It needs **Delete Computer
  objects** on the OUs the servers live in, as well as the usual Create. A bare user name gets `@<domain_name>`
  when `domain_name` is set; otherwise use `DOMAIN\user` or `user@domain`.
- **Connecting to the host:** the job's Machine credential is the Proxmox SSH login, so the WinRM login to
  `ad_cleanup_host` comes from its inventory variables. Either:
  - use a member server (not a DC) and add the domain-join account to that server's local **Remote Management
    Users** group, then set its `ansible_user` / `ansible_password` to `{{ domain_join_user }}` /
    `{{ domain_join_pass }}`: one account for everything; or
  - connect with a separate account allowed WinRM on the host (e.g. a DC), from a custom credential type that
    injects `ad_management_user` / `ad_management_password`, as in the example inventory.

  The execution environment needs `pypsrp` ([requirements.txt](requirements.txt)) and the `microsoft.ad`
  collection ([requirements.yml](requirements.yml)).

| Variable | Default | Purpose |
| --- | --- | --- |
| `ad_cleanup` | `true` | Delete Windows VMs' AD computer objects. |
| `ad_cleanup_host` | `""` | Inventory host that runs the delete; `""` skips it (with a warning per Windows VM). |
| `ad_cleanup_user` / `ad_cleanup_password` | `domain_join_user` / `domain_join_pass` | Account that deletes the object. |
| `ad_cleanup_domain_server` | `""` | Domain controller to use (`""` = the ActiveDirectory module picks one). |
| `ad_computer_name` (per VM) | the VM's Proxmox name | Computer account to delete. |
| `os_type` (per VM) | from the VM's `ostype` | `windows` or `linux`. |

## Development and CI

Two workflows call reusable workflows from
[`SAL9000-HomeLab/shared-actions`](https://github.com/SAL9000-HomeLab/shared-actions):

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on every pull request:
  - `yamllint`, then `ansible-playbook --syntax-check` on `site.yml`.
  - `ansible-lint` using [`.ansible-lint`](.ansible-lint). The only rule skipped is
    `var-naming[no-role-prefix]`: the role's variables (`vms_to_delete`, `proxmox`, `netbox`,
    `netbox_cleanup`, `technitium_dns`, …) are set by AWX job templates and inventories, so
    prefixing them with `proxmox_deprovision_` would break existing callers.
- **Linting Validation** (`.github/workflows/ci.yml`), on pull requests to `main`:
  - Markdown lint (`markdownlint-cli2`) using [`.markdownlint.json`](.markdownlint.json):
    120-column lines (code blocks and tables exempt), `_emphasis_` and `**strong**`.
  - Link check (linkspector) using [`.linkspector.yml`](.linkspector.yml). Findings are
    reported on the pull request. Links to this org's GitHub repos are skipped.
  - `yamllint` again, standalone.

Both YAML checks read [`.yamllint.yml`](.yamllint.yml): the default rules with 120-column
lines (matching the editor ruler in `.vscode/settings.json`), with truthy checks skipped for
GitHub workflows (`on:`). The file must keep the `.yml` name, because the shared `lint-yaml`
workflow loads it by that exact path.

Run the same checks locally before opening a pull request:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "yamllint>=1.30" "ansible>=2.15" "ansible-lint>=6"
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-playbook -i localhost, -c local --syntax-check site.yml
ansible-lint .
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Project words for the VS Code spell checker live in `.vscode/cspell.json`.

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a
`vX.Y.Z` tag publishes that version's section as a GitHub release.
