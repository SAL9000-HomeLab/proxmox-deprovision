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

## Development and CI

Two workflows call reusable workflows from
[`SAL9000-HomeLab/shared-actions`](https://github.com/SAL9000-HomeLab/shared-actions):

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on pushes to `main` and every pull request:
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
