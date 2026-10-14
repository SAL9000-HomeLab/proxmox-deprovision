# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

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
