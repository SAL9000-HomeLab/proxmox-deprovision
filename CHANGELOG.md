# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Fixed: a VM on another node than `proxmox.node` couldn't be deleted ("not found on node"; `qm` only sees its own
  node's VMs), and a wrong `vmid` deleted whatever VM had it. The VMID is now looked up cluster-wide and deleted on
  its own node, and the job stops before any change when the VMID is a container or a template or belongs to a VM
  with another name than the entry's `name`. With several hosts in the `proxmox` group, the list is processed once
  (it ran once per host).
- Fixed: NetBox cleanup deletes the records of every address the VM has in an `ipconfigN`, not only `ipconfig0`, so
  the addresses proxmox-deploy reserves for extra network interfaces (`extra_nics`) are released too.

- Added: Windows VMs (Proxmox `ostype` win*, or `os_type: windows`) have their Active Directory computer object
  deleted after the VM is destroyed, with `microsoft.ad.computer` on `ad_cleanup_host` (a DC or a server with
  RSAT-AD-PowerShell), as the domain-join account by default. Skipped with a warning when `ad_cleanup_host` isn't set.
- Fixed: deprovisioning a VM without a static `ipconfig0` (DHCP, or a deploy that failed before it got an IP) failed
  with a templating error; it now skips the address-based DNS cleanup.
- Fixed: NetBox v2 API tokens (`nbt_<key>.<token>`, the default since NetBox 4.5) are sent as `Bearer`; they were
  sent as `Token`, which NetBox rejects with "Invalid authorization header". v1 tokens still use `Token`.
- Added: CI via the shared `SAL9000-HomeLab/shared-actions` workflows: Ansible checks (yamllint,
  syntax check, ansible-lint) on pushes to `main` and pull requests, and Markdown, link and YAML
  linting on pull requests. Adds `.yamllint.yml`, `.ansible-lint`, `.markdownlint.json`,
  `.linkspector.yml`, a PR template, a tag-driven release workflow and this changelog.
- Changed: Tasks use FQCN module names and wrap at 120 columns. The VM stop task now reports
  `changed` when it stops a VM.
- Fixed: DNS cleanup logs and continues when the VM has no Technitium PTR or A record, instead of
  failing the job. (PR #4)
- Fixed: NetBox cleanup deletes only records that match the VM exactly. It previously deleted
  every result of a substring search, so `web-01` also removed `web-010`. (PR #3)
- Added: The VM's `user-data-<vmid>` / `cloudbase-<vmid>` snippets are removed from the node. (PR #3)
- Removed: The unused `destroy_disk` option; `qm destroy --purge` always deletes the disks. (PR #3)
- Changed: Ansible CI runs on pull requests only, no longer on pushes to `main` (synced from ans-template).
