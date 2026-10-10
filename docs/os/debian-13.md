# Debian 13 deployment notes

`shard-manifest` runs as a stock systemd service under Debian 13, deployed by
the same playbook as [Ubuntu 24.04](ubuntu-24.04.md) (both are
`ansible_os_family: Debian`); prerequisites, deploy, update and firewall
procedures on that page apply unchanged. Platform differences are in the
canonical [Debian 13 notes](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/os/debian-13.md).
