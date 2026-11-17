# Changelog

## Unreleased
- Fix: `--check` with `remediate` no longer fails on a non-compliant host (the re-audit is skipped in check mode).
- Fix: works with `gather_facts: false` (gathers the minimum facts itself).
- Fix: findings report `actual: unset`/`unknown` instead of `null` for a missing `SELINUX=` or an unknown boolean.
- Docs: README example uses the standalone role name `iamenr0s.selinux_hardening`; added Requirements and Role Variables.
- Change: booleans are set with `ansible.posix.seboolean` (new `ansible.posix` dependency; installs `python3-libselinux`/`python3-libsemanage` when booleans are set).
- Internal: simpler config and boolean parsing; CI lint runs the pinned pre-commit hooks.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
