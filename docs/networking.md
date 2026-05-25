# Networking

`bitcoin-shard-manifest` requires only outbound IPv6 multicast egress and
inbound TCP for Prometheus scraping.

## Egress

Manifest datagrams are sent to one or more of:

| Scope    | Address          |
| -------- | ---------------- |
| `link`   | `FF02::B:FFFD`   |
| `site`   | `FF05::B:FFFD`   |
| `org`    | `FF08::B:FFFD`   |
| `global` | `FF0E::B:FFFD`   |

The destination port is `manifest_port` (default `9001`, matching the
existing beacon listen port). The 16-bit IANA group-id occupying bytes
[12:14] is `mc_group_id` (default `0x000B`).

The daemon binds the egress socket to the interface named by `iface`; when
`iface` is empty the first non-loopback interface with a global IPv6
address is selected. **Set `iface` explicitly in production**.

## Ingress

Only Prometheus scraping needs reachability. The default `metrics_addr`
binds `[::]:9091`. The firewall role allows TCP port 9091 from
`mgmt_cidrs_v4` / `mgmt_cidrs_v6`.

There is no NACK / ACK / response traffic for BRC-137; manifests are pure
fire-and-forget multicast.

## Multicast routing notes

- The host kernel must have IPv6 multicast on the chosen interface.
- For `site` scope across an L3 boundary, the upstream router must run PIM
  or equivalent for the FF05::/16 scope.
- For `global` scope, ensure the upstream provider does not filter ff0e::
  destinations.
- MLD snooping on switches is harmless: announcements are unidirectional
  egress; the local node does not need to subscribe.

## Coexistence with BRC-126 ADVERT

BRC-126 retry-endpoint ADVERTs (MsgType `0x20`) and BRC-137 ShardManifests
(MsgType `0x40`) share the beacon group. Listeners on the beacon group
distinguish them by `buf[6]`. There is no port conflict and no risk of
delivery interference.
