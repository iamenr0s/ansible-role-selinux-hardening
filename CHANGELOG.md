# Changelog

## Unreleased
- Change: removed `selinux_hardening_config`; SELinux only reads `/etc/selinux/config`, so the path is fixed.
- Change: the fail-on-findings decision is two named tasks instead of one nested expression.
- Change: task names in sub-task files are prefixed (`audit |`, `facts |`, `configure |`).
- Change: remediate gathers the SELinux state once less (only package facts before the install).
- Change: pre-commit runs gitleaks secret detection.
- Docs: README intro aligned with the sibling hardening roles; the example uses `become: true`.
- Change: variables renamed to the role prefix `selinux_hardening_*` (internal: `__selinux_hardening_*`); the `var-naming[no-role-prefix]` lint check is enabled again.
- Fix: `--check` with `remediate` no longer fails on a non-compliant host (the re-audit is skipped in check mode).
- Fix: works with `gather_facts: false` (gathers the minimum facts itself).
- Fix: findings report `actual: unset`/`unknown` instead of `null` for a missing `SELINUX=` or an unknown boolean.
- Docs: README example uses the standalone role name `iamenr0s.selinux_hardening`; added Requirements and Role Variables.
- Change: booleans are set with `ansible.posix.seboolean` (new `ansible.posix` dependency; installs `python3-libselinux`/`python3-libsemanage` when booleans are set).
- Change: audit prints one line per finding (`key: expected X, got Y`); `selinux_hardening_findings` keeps its structure.
- Internal: audit written as one task per check; package lists defined once in `vars/main.yml`; simpler config and boolean parsing; CI lint runs the pinned pre-commit hooks.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
