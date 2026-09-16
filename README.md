# ansible-playbook-rhel

[![Linter](https://github.com/thbe/ansible-playbook-rhel/actions/workflows/linter.yml/badge.svg)](https://github.com/thbe/ansible-playbook-rhel/actions/workflows/linter.yml)

Red Hat Enterprise Linux (RHEL) lifecycle playbooks: package/patch management with
`dnf`, Red Hat Insights integration, subscription and RHC registration, Leapp
in-place upgrade preparation, and cloud (Azure/AWS) OS-disk provisioning.

This repository is normally consumed as the `playbooks/rhel` git submodule of
[ansible-main](https://github.com/thbe/ansible-main), but it can also be used
standalone.

## Table of Contents

- [Requirements](#requirements)
- [Playbooks](#playbooks)
  - [Patch management (dnf)](#patch-management-dnf)
  - [Red Hat Insights](#red-hat-insights)
  - [Subscription / RHC](#subscription--rhc)
  - [Leapp upgrade preparation](#leapp-upgrade-preparation)
  - [Cloud OS disks](#cloud-os-disks)
- [Usage](#usage)
- [License](#license)
- [Author](#author)

## Requirements

- Ansible 2.14+ (core) with the collections:
  - `ansible.posix`
  - `community.general`
- Target hosts running RHEL 8 or 9 (most tasks are gated on
  `os_family == "RedHat"` and major version >= 8).
- Valid Red Hat subscription credentials for the registration playbooks.
- Privilege escalation (`become`) available. All playbooks target `all`.

## Playbooks

### Patch management (dnf)

| Playbook                    | Description                                                                    |
| --------------------------- | ------------------------------------------------------------------------------ |
| `dnf_autoremove.yml`        | Refresh cache and remove orphaned/unused packages.                             |
| `dnf_upgrade.yml`           | Full package upgrade. Honours `rhel_release_version`, `dnf_skip_broken`, `dnf_nobest`. |
| `dnf_upgrade_no_kernel.yml` | Full upgrade but excludes `kernel*` packages.                                   |
| `dnf_upgrade_nogpg.yml`     | Full upgrade with GPG signature checking disabled (use only for trusted repos).|
| `dnf_upgrade_security.yml`  | Applies security errata only.                                                  |

Key variables (defaults in the playbooks): `rhel_release_version` (`latest`),
`dnf_skip_broken` (`false`), `dnf_nobest` (`false`), `dnf_exclude` (`kernel*`).

### Red Hat Insights

All Insights playbooks wrap `/usr/bin/insights-client` and are safe to re-run
(`changed_when: false`).

| Playbook                  | Description                                       |
| ------------------------- | ------------------------------------------------- |
| `insights_register.yml`   | Register the host with Red Hat Insights.          |
| `insights_check_in.yml`   | Lightweight check-in.                             |
| `insights_run.yml`        | Standard Insights collection run.                 |
| `insights_compliance.yml` | Run the compliance collector.                     |
| `insights_malware.yml`    | Run the malware-detection collector.              |
| `insights_full_run.yml`   | Check-in + run + compliance + malware in sequence.|

### Subscription / RHC

| Playbook                        | Description                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------- |
| `subscription_registration.yml` | Register via `community.general.redhat_subscription`, selecting a syspurpose profile by activation key (`Developer-RHEL`, `Standard-RHEL`, `Standard-SAP-RHEL`, `Premium-SAP-RHEL`). |
| `subscription_refresh.yml`      | Refresh subscription data (`subscription-manager refresh`).                                        |
| `rhc_register.yml`              | Install and connect the Remote Host Configuration (`rhc`) client. Requires `rhn_activation_key`, `rhn_organization_id`. |
| `rhc_status.yml`                | Report the current `rhc` connection status.                                                       |

Required variables: `rhn_activation_key`, `rhn_organization_id` (defaults to
`unset` where applicable — supply via inventory or `--extra-vars`).

### Leapp upgrade preparation

| Playbook            | Description                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| `prepare_leapp.yml` | Prepare a RHEL 8 host for an in-place major upgrade: enable base repos, pin the release, versionlock, full upgrade, and install `leapp-upgrade`/`cockpit-leapp`. |

### Cloud OS disks

The `disks/` subtree provisions LVM layouts on cloud VMs.

| Playbook                  | Description                                                               |
| ------------------------- | ------------------------------------------------------------------------- |
| `disks/azure/os_setup.yml`| Build the `rootvg` volume group and root/usr/var/home/tmp/swap LVs on Azure. |
| `disks/aws/os_setup.yml`  | Equivalent layout for AWS. *(Note: currently references Azure device paths — see the audit notes.)* |

## Usage

```shell
# Register a new host, then patch and reboot it
ansible-playbook -i inventories/prod/hosts.yml subscription_registration.yml \
  --extra-vars 'rhn_activation_key=Standard-RHEL rhn_organization_id=123456' --limit new01
ansible-playbook -i inventories/prod/hosts.yml insights_register.yml --limit new01
ansible-playbook -i inventories/prod/hosts.yml dnf_upgrade.yml --limit new01

# Apply security errata only, fleet-wide
ansible-playbook -i inventories/prod/hosts.yml dnf_upgrade_security.yml

# Prepare a RHEL 8 host for a Leapp upgrade
ansible-playbook -i inventories/prod/hosts.yml prepare_leapp.yml --limit legacy01
```

## License

GPL-3.0-only

## Author

Thomas Bendler - [https://www.thbe.org/](https://www.thbe.org/)
