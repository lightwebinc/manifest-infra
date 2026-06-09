# Architecture

`manifest-infra` deploys [`shard-manifest`](https://github.com/lightwebinc/shard-manifest)
to one or more hosts. Each host runs the daemon as an unprivileged systemd
(or rc.d) service and periodically emits BRC-139 ShardManifest datagrams to
the IPv6 multicast beacon group.

## Per-host deployment

```
+------------------------------------------------------------+
|  Host (Ubuntu 24.04 / Debian / FreeBSD)                    |
|                                                            |
|  +-----------------------+    +-----------------------+    |
|  | systemd / rc.d        |    | nftables / pf         |    |
|  | shard-manifest|    | (perimeter firewall)  |    |
|  +-----------+-----------+    +-----------------------+    |
|              |                                             |
|              | UDP egress (multicast)                      |
|              v                                             |
|  +-----------+-----------+                                 |
|  | egress NIC            |                                 |
|  +-----------+-----------+                                 |
|              |                                             |
+--------------|---------------------------------------------+
               v
         IPv6 multicast fabric
         FF0X::B:FFFD (beacon group)
```

## Roles

### `common`
- Installs Go toolchain (`/usr/local/go`), build dependencies (`build-essential`, `git`, `acl`, `curl`, `tar`, `ca-certificates` on Debian; the FreeBSD equivalents under FreeBSD).
- Provides an opt-in OS package upgrade task tagged `os_update` (skipped by default).

### `shard-manifest`
- Creates the `manifest-infra` system user/group.
- Clones (or fetches) the daemon source into `/opt/shard-manifest`.
- Builds with `go build -buildvcs=false` for the host's GOOS/GOARCH (or
  installs a pre-built binary via `manifest_local_binary`).
- Renders `/etc/shard-manifest/config.env` (Linux) or
  `/usr/local/etc/shard-manifest.conf` (FreeBSD) from the deployment
  variables.
- Installs and enables the systemd unit (Linux) or rc.d script (FreeBSD)
  with hardening (`NoNewPrivileges`, `ProtectSystem=strict`,
  `RestrictAddressFamilies`, empty `CapabilityBoundingSet`).

### `firewall` (optional, default on)
- Linux: nftables ruleset under `inet shard-manifest` allowing
  - UDP egress to `ff05::/16`, `ff08::/16`, `ff0e::/16` on the manifest port,
  - SSH + Prometheus scrape from `mgmt_cidrs_*`,
  - Standard ICMPv6, NDP, MLD, DHCPv6, DNS, NTP, HTTPS,
  - OTLP gRPC/HTTP egress when `otlp_endpoint` is set.
- FreeBSD: equivalent pf anchor under `shard-manifest`.

## Failure model

The daemon is purely a sender. It does not subscribe to or interpret any
traffic. Common failure modes:

| Symptom                            | Likely cause                                | Remedy                                                   |
| ---------------------------------- | ------------------------------------------- | -------------------------------------------------------- |
| `/readyz` 503 stale                | Multicast egress dropped                    | Inspect `nftables -L` / `pfctl -sa`; verify route/iface  |
| Service flaps                      | `iface` invalid or no global IPv6 address   | Set `iface` per host; check `ip -6 addr show`            |
| `bsm_send_errors_total{kind=write}` rising | Upstream MTU drop or firewall            | Check ICMPv6 PTB, MLD snooping on switch                |
| Manifest never reaches consumers   | Wrong `manifest_scope` / `mc_group_id`      | Align with the rest of the network (BRC-129)            |
