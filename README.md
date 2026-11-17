[![Molecule](https://github.com/iamenr0s/ansible-role-selinux-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-selinux-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/selinux_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-selinux-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-selinux-hardening)

# Ansible Role: Selinux Hardening

> Standalone copy of `selinux_hardening` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables use the role prefix `selinux_hardening_*` (the collection uses `linux_hardening_*`).

Audit-first SELinux for the RedHat family (EL, Fedora). Other families are **skipped** with a message and `selinux_hardening_findings: []` (Debian/Ubuntu: see `apparmor_hardening`, not yet written). The role **never reboots**.

## What it checks
| Check | Remediation |
|---|---|
| `selinux-policy-targeted`, `libselinux-utils`, `policycoreutils` installed | installed |
| `SELINUX=` in `/etc/selinux/config` equals `selinux_hardening_state` (default `enforcing`) | written (backup kept) |
| Running mode equals the desired state | `setenforce` |
| Persistent booleans in `selinux_hardening_booleans` | `ansible.posix.seboolean` (persistent) |
| `SELINUXTYPE` equals `targeted` | **reported only** (changing it needs a relabel) |
| `selinux=0` / `enforcing=0` on the kernel command line | **reported only** (bootloader) |

## The three starting points
- **Enforcing already** (stock EL images): nothing to do beyond drift and booleans.
- **Permissive**: raising to enforcing can break services that relied on it, so remediation stops unless `selinux_hardening_confirm_enforcing: true`. Check `ausearch -m avc -ts boot` first.
- **Disabled**: enabling needs a full relabel and a reboot. With `selinux_hardening_allow_enable_from_disabled: true` the role **stages** it: config set to *permissive* (never straight to enforcing), `/.autorelabel` created. Then you: remove `selinux=0` if it is set (`grubby --update-kernel ALL --remove-args selinux=0`), reboot, let the relabel finish, and run the role again with `confirm_enforcing: true`. The staged state is reported as a non-fixable finding until then.

## Requirements
Run as root (`become: true`). Needs the `ansible.posix` collection (`ansible-galaxy collection install -r requirements.yml`; already included in the full `ansible` package). When booleans are set, the role installs `python3-libselinux` and `python3-libsemanage` on the host. The role is idempotent: a second remediate run reports no changes.

## Role Variables
| Variable | Default | Description |
|---|---|---|
| `selinux_hardening_mode` | `audit` | `audit` (read-only) or `remediate` |
| `selinux_hardening_confirm_remediation` | `false` | Must be `true` before remediate changes anything |
| `selinux_hardening_backup` | `true` | Back up `/etc/selinux/config` when it changes |
| `selinux_hardening_verify` | `true` | Re-audit after remediation (skipped in check mode) |
| `selinux_hardening_fail_on_error` | `true` | Fail the play when audit finds deviations |
| `selinux_hardening_enabled` | `true` | Run the role |
| `selinux_hardening_config` | `/etc/selinux/config` | SELinux configuration file |
| `selinux_hardening_state` | `enforcing` | Desired mode: `enforcing` or `permissive` |
| `selinux_hardening_policy` | `targeted` | Expected policy type (reported only) |
| `selinux_hardening_confirm_enforcing` | `false` | Allow raising the running mode from permissive to enforcing |
| `selinux_hardening_allow_enable_from_disabled` | `false` | Allow staging the enable from disabled (config permissive plus `/.autorelabel`; reboot yourself) |
| `selinux_hardening_booleans` | `{}` | Persistent booleans `{name: true\|false}` |

The full spec is in `meta/argument_specs.yml`.

## Modes
`audit` (default) is read-only and sets `selinux_hardening_findings` (`key, expected, actual, fixable`). `remediate` needs `selinux_hardening_confirm_remediation: true`; after it only fixable findings fail the run.

## Example
```yaml
- hosts: rhel_family
  roles:
    - role: iamenr0s.selinux_hardening
      vars:
        selinux_hardening_mode: remediate
        selinux_hardening_confirm_remediation: true
        selinux_hardening_confirm_enforcing: true
        selinux_hardening_booleans: {httpd_can_network_connect: true}
```

## Verification
Container scenarios `default` (the staged enable at file level: containers read as disabled), `audit`, `refuse` (no confirmation; no consent to stage); Debian-family containers check the skip. **VM scenarios** (libvirt, RedHat boxes only; run `sg libvirt -c "molecule test -s vagrant"`): `vagrant` (permissive host: refuses without consent, then enforces and sets a boolean) and `vagrant-relabel` (really disabled via `selinux=0`, staged, relabel reboot, permissive, second run enforces, ~2 minutes). Tested: AlmaLinux 9 (containers and VMs), Fedora 44 (containers). CI runs the container scenarios only.

## Rollback
Restore `/etc/selinux/config` from its backup and `setenforce 0`. To undo a boolean, set it back (`setsebool -P <name> <on|off>`).

## Known gaps
- Does not audit or fix file contexts, unconfined services, `setroubleshoot`/`mcstrans` packages, or custom policy modules.
- `SELINUXTYPE` is only reported; `mls` and custom policies are untested.
- A relabel on a large filesystem can take a long time; plan the reboot window.
- Not tested on Fedora VMs, EL8/10, or with `auditd` installed (the AVC check is a documented manual step). CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-selinux-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-selinux-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.selinux_hardening`
