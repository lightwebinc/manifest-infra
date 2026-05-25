# Security

## Service hardening

The systemd unit applies:

- `User=` / `Group=` — runs as `bitcoin-manifest` (system user, nologin shell).
- `NoNewPrivileges=true`
- `PrivateTmp=true`
- `ProtectSystem=strict`
- `ProtectHome=true`
- `ProtectKernelTunables=true`
- `ProtectKernelModules=true`
- `ProtectControlGroups=true`
- `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX AF_NETLINK`
- `CapabilityBoundingSet=` (empty)
- `AmbientCapabilities=` (empty)
- `LockPersonality=true`
- `SystemCallArchitectures=native`

The daemon needs no special privileges: it binds an unprivileged UDP socket
and an unprivileged HTTP port (default 9091). Running rootless is the
default.

## Firewall

The default ruleset:

- Drops by default on `input` and `output`.
- Allows established/related state.
- Allows SSH (TCP 22) and Prometheus scrape (`metrics_port`, default 9091)
  from `mgmt_cidrs_v4` / `mgmt_cidrs_v6` only.
- Allows IPv6 multicast egress to `ff05::/16`, `ff08::/16`, `ff0e::/16` on
  `manifest_port` (default 9001).
- Allows ICMPv6, NDP, MLD, DHCPv6 client.
- Allows outbound DNS, NTP, HTTPS for package mirrors and time sync.
- Allows OTLP gRPC/HTTP egress (TCP 4317/4318) when `otlp_endpoint` is set.

Set `enable_firewall: false` to skip the role on hosts where another
firewall management system owns the perimeter.

## Authoritative announcements

Set `authoritative: true` on a small operator-curated set of manifest
emitters (typically tied to your orchestration source-of-truth). Operators
SHOULD treat non-authoritative manifests as observational only and rely on
the authoritative set when (in future revisions) automated shard-bit
shifts are enabled. See [BRC-137 Safety Guidance](https://github.com/lightwebinc/bitcoin-multicast/blob/main/docs/brc-137-shard-manifest.md#safety-guidance-non-normative).

## Identity

`InstanceID` is `CRC32c(hostname)`. It is stable across restarts but is
**not** cryptographically authenticated. Consumers in a future revision
SHOULD pair manifests with operator-managed allow-lists or signed envelopes
before acting on them automatically. The protocol leaves room for this via
the `Reserved` and `Flags` bytes.
