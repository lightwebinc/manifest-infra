# manifest-infra

> Part of the [**BSV Layered Multicast**](https://github.com/lightwebinc/bsv-multicast) open-source project — see the main repository for the full architecture, design docs, and BRC specifications.

Deployment automation for [`shard-manifest`](https://github.com/lightwebinc/shard-manifest)
nodes — the BRC-139 Shard Manifest Announcement Daemon.

This repository is the VM-side counterpart to
[`shard-manifest-helm`](https://github.com/lightwebinc/shard-manifest-helm)
(the Kubernetes chart). It contains:

- An **Ansible** playbook + roles to install and configure the daemon on
  Ubuntu 24.04 / Debian 12 / FreeBSD 14 hosts.
- A perimeter **firewall** role (nftables on Linux, pf on FreeBSD).
- A **common** role that installs the Go toolchain and base build deps.
- Documentation under [`docs/`](docs/).

No terraform, no BGP, no special networking: the manifest service is a tiny
stateless emitter and runs anywhere with IPv6 multicast egress.

## Quick start

```bash
# Install collections.
cd ansible
ansible-galaxy collection install -r requirements.yml

# Copy + edit the example inventory.
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml

# Deploy.
ansible-playbook -i inventory/hosts.yml site.yml
```

See [`docs/ansible.md`](docs/ansible.md) for variable reference and
[`docs/architecture.md`](docs/architecture.md) for the deployment model.

## Layout

```
.
├── ansible/
│   ├── site.yml
│   ├── requirements.yml
│   ├── group_vars/all.yml
│   ├── inventory/hosts.example.yml
│   └── roles/
│       ├── common/                       # base packages + Go toolchain
│       ├── firewall/                     # nftables (Linux) + pf (FreeBSD)
│       └── shard-manifest/               # daemon install + systemd / rc.d
├── docs/
│   ├── architecture.md
│   ├── ansible.md
│   ├── networking.md
│   ├── security.md
│   └── os/
│       ├── ubuntu-24.04.md
│       └── freebsd-14.md
├── .github/workflows/lint.yml
├── .yamllint.yml
├── .gitignore
├── LICENSE
└── README.md
```

## License

Apache 2.0. See [LICENSE](LICENSE).
