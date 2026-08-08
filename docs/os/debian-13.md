# Debian 13 deployment notes

`shard-manifest` runs as a stock systemd service under Debian 13, deployed by
the same playbook as [Ubuntu 24.04](ubuntu-24.04.md) — both are
`ansible_os_family: Debian`, so the Ansible roles run the same task set.

## Differences from Ubuntu

- **Minimal images.** Debian cloud images omit `curl`; the `common` role
  installs it with the build dependencies before the Go toolchain download
  needs it.
- **Kernel.** The stock Debian 13 kernel (6.12 LTS) is sufficient; no
  backports kernel is required.
- Prerequisites, deploy, update, and firewall-disable procedures are the same
  as the Ubuntu page.
