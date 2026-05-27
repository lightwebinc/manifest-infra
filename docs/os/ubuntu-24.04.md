# Ubuntu 24.04 deployment notes

`shard-manifest` runs as a stock systemd service under Ubuntu 24.04.

## Prerequisites

- An IPv6 default route or scope-appropriate IPv6 route on the egress
  interface.
- nftables installed (the role installs `nftables` via apt; `iptables-nft`
  alternative also works).
- Outbound port 443 reachable for the initial Go toolchain download (or
  pre-stage `/usr/local/go`).

## Deploy

```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml
```

After the play completes:

```bash
systemctl status shard-manifest
journalctl -u shard-manifest -n 50
curl -s http://[::1]:9091/healthz
curl -s http://[::1]:9091/readyz
curl -s http://[::1]:9091/metrics | grep '^bsm_'
```

## Updating

```bash
# Bump manifest_version to a tag and re-run.
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags manifest
```

The build is idempotent. Pass `manifest_force_build=true` to rebuild when
the source already matches.

## Disabling the firewall role

```bash
ansible-playbook ... --skip-tags firewall
# or
ansible-playbook ... -e enable_firewall=false
```
