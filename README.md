# manifest-infra

[![Lint](https://github.com/lightwebinc/manifest-infra/actions/workflows/lint.yml/badge.svg)](https://github.com/lightwebinc/manifest-infra/actions/workflows/lint.yml)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

> Part of the [**BSV Layered Multicast**](https://github.com/lightwebinc/bsv-multicast) open-source project; see the main repository for the full architecture, design docs, and BRC specifications.

Ansible automation for deploying
[`shard-manifest`](https://github.com/lightwebinc/shard-manifest)
nodes, the BRC-139 shard manifest announcer. It is the host-deployment
counterpart of the
[`charts/shard-manifest`](https://github.com/lightwebinc/charts/tree/main/charts/shard-manifest)
Kubernetes chart.

```text
shard-manifest  ──multicast──▶  FF05::B:FFFD (beacon group)
(this repo deploys)
```

No Terraform, no BGP, no special networking: the manifest service is a tiny
stateless emitter and runs anywhere with IPv6 multicast egress. A perimeter
firewall role (nftables / pf) is included.

## Platforms

Ubuntu 24.04, Debian 13 and FreeBSD 14 via Ansible. See
[supported platforms](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/platforms.md).

## Quick Start

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml
ansible-playbook -i inventory/hosts.yml site.yml
```

## Documentation

- [Shared host-deployment docs](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/README.md) (platforms, Ansible operations, OS notes)
- [Architecture](docs/architecture.md)
- [Ansible usage](docs/ansible.md)
- [Security (perimeter firewall)](docs/security.md)
- [Networking](docs/networking.md)
- OS notes: [Ubuntu 24.04](docs/os/ubuntu-24.04.md), [Debian 13](docs/os/debian-13.md), [FreeBSD 14](docs/os/freebsd-14.md)

## Repository Layout

```text
ansible/     Roles and playbooks
docs/        Per-topic documentation
```

## License

Apache 2.0; see [LICENSE](LICENSE).
