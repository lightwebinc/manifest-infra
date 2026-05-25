# Ansible

The playbook ships three roles applied to the `manifest_nodes` group:

1. `common` — packages + Go toolchain.
2. `bitcoin-shard-manifest` — daemon install, config render, service unit.
3. `firewall` — perimeter ruleset (Linux nftables, FreeBSD pf).

Run it with:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml
```

## Variables

All variables are documented inline in
[`ansible/group_vars/all.yml`](../ansible/group_vars/all.yml). The most
commonly overridden ones are summarised below.

### Required per-deployment

| Variable        | Default | Notes                                                                |
| --------------- | ------- | -------------------------------------------------------------------- |
| `shard_bits`    | `0`     | MUST match the rest of the network. 0..12 per BRC-129.               |
| `mgmt_cidrs_v4` | `[]`    | SSH + Prometheus scrape allow-list. MUST be non-empty when firewall is on. |
| `iface`         | `""`    | Egress interface; SHOULD be set per-host to avoid auto-pick surprises. |

### Manifest content

| Variable             | Default     | Notes                                                       |
| -------------------- | ----------- | ----------------------------------------------------------- |
| `joined_groups`      | `""`        | Comma list of indices, or `all`, or empty (identity-only).  |
| `manifest_encoding`  | `auto`      | `auto`/`list`/`bitmap`.                                     |
| `role_hint`          | `generic`   | Informational; one of generic/proxy/listener/retry-endpoint/producer/manifest-only. |
| `generation_id`      | `""`        | 16-byte hex; bump when `shard_bits` changes.                |
| `authoritative`      | `false`     | Sets `Flags.Authoritative` on the wire.                     |

### Cadence

| Variable             | Default | Notes                                                |
| -------------------- | ------- | ---------------------------------------------------- |
| `announce_interval`  | `300s`  | Send period; jittered ±10 % at runtime.              |
| `ttl`                | `0s`    | Wire-format TTL; `0` = consumer default.             |

### Network

| Variable          | Default     | Notes                                       |
| ----------------- | ----------- | ------------------------------------------- |
| `manifest_port`   | `9001`      | UDP destination port.                       |
| `manifest_scope`  | `site`      | Comma list of `link,site,org,global`.       |
| `mc_group_id`     | `0x000B`    | IANA group-id (BRC-129).                    |

### Observability

| Variable          | Default        | Notes                                            |
| ----------------- | -------------- | ------------------------------------------------ |
| `metrics_addr`    | `[::]:9091`    | HTTP listener.                                   |
| `metrics_port`    | `9091`         | MUST match `metrics_addr`; used by firewall rules. |
| `otlp_endpoint`   | `""`           | Optional OTLP gRPC endpoint.                     |

## Tags

| Tag         | What it runs                                                  |
| ----------- | ------------------------------------------------------------- |
| `common`    | Base packages + Go toolchain.                                 |
| `manifest`  | Daemon install + config + service.                            |
| `firewall`  | Render and reload the perimeter ruleset.                      |
| `os_update` | Opt-in package upgrade (`--tags os_update`).                  |

## Variable precedence reminder

Ansible's `group_vars/all` overrides inventory-group `vars:`. Set host-level
overrides under `hosts:` rather than `vars:` when they need to win against
group_vars/all (e.g. `iface`, `role_hint`).

## Pushing a pre-built binary (skip git/build)

```yaml
manifest_local_binary: "/path/to/local/bitcoin-shard-manifest"
```

When set, the role copies the binary in place of cloning + building. Useful
for air-gapped deployments and during development.
