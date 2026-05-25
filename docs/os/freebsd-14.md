# FreeBSD 14 deployment notes

`bitcoin-shard-manifest` runs as a stock rc.d service under FreeBSD 14.

## Prerequisites

- pf enabled (`/etc/rc.conf`: `pf_enable="YES"`). The `firewall` role takes
  care of this when applied.
- Outbound IPv6 reachable on the egress interface.
- `pkg` access for `git`, `bash`, and the Go toolchain (or pre-stage
  `/usr/local/go`).

## Deploy

```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml
```

After the play completes:

```bash
service bitcoin_shard_manifest status
tail -n 50 /var/log/bitcoin_shard_manifest.log
fetch -qo - http://[::1]:9091/healthz
fetch -qo - http://[::1]:9091/metrics | grep '^bsm_'
```

## Updating

Bump `manifest_version` and re-run with `--tags manifest`. The rc.d script
is restarted via the `Restart bitcoin-shard-manifest` handler.

## Notes

- The rc.d script reads the rendered config at
  `/usr/local/etc/bitcoin-shard-manifest.conf` and exports each variable
  before invoking `/usr/sbin/daemon` to fork the binary.
- The pf anchor is loaded from `/etc/pf.anchors/bitcoin-shard-manifest`
  via a `load anchor` directive in `/etc/pf.conf`. Reload pf manually with
  `pfctl -f /etc/pf.conf` if you bypass Ansible.
