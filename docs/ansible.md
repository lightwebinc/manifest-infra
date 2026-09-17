# Ansible

The playbook ships three roles applied to the `manifest_nodes` group:

1. `common` — packages + Go toolchain, journald cap + disk-reclaim timer (Linux), opt-in `--tags os_update` patching.
2. `shard-manifest` — daemon install, config render, service unit.
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
| `shard_bits`    | `2`     | MUST match the rest of the network. 0..12 per BRC-129.               |
| `mgmt_cidrs_v4` | `[]`    | SSH + Prometheus scrape allow-list. The play fails when the firewall is on and both `mgmt_cidrs_v4` / `mgmt_cidrs_v6` are empty. |
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

### SSM data-plane advertisement

| Variable                      | Default | Notes                                                                    |
| ----------------------------- | ------- | ------------------------------------------------------------------------ |
| `manifest_source_mode`        | `asm`   | `asm`/`ssm`. `ssm` REQUIRES `manifest_publishers` to be non-empty.       |
| `manifest_publishers`         | `""`    | Comma list of data-plane publisher IPv6 addresses or DNS names (SSM Sources payload). |
| `manifest_publishers_refresh` | `30s`   | DNS re-resolve interval for `manifest_publishers` entries; must be `> 0`. |

### Generation rollover (Successor block)

Leave `manifest_successor_generation_id` empty to disable. When set,
`manifest_successor_transition_epoch` is REQUIRED and must be at least
`now + 2 × announce_interval`. `manifest_successor_shard_bits` must differ from
`shard_bits` by ±1. The successor env vars are only rendered into `config.env`
when `manifest_successor_generation_id` is non-empty.

| Variable                              | Default | Notes                                                              |
| ------------------------------------- | ------- | ------------------------------------------------------------------ |
| `manifest_pilot_only`                 | `false` | Sets `Flags.PilotOnly`: the manifest describes desired fleet state, not the node's own joins (implies `authoritative`). Set on the pilot node driving a rollover. |
| `manifest_successor_generation_id`    | `""`    | Incoming generation 16-byte hex; empty = no Successor block.       |
| `manifest_successor_shard_bits`       | `0`     | Incoming generation shard bits; must differ from `shard_bits` by ±1. |
| `manifest_successor_source_mode`      | `""`    | Incoming addressing model `asm`/`ssm`; empty = inherit `manifest_source_mode`. |
| `manifest_successor_transition_epoch` | `0`     | Unix seconds at which the successor becomes the sole active generation. |

### Observability

| Variable          | Default        | Notes                                            |
| ----------------- | -------------- | ------------------------------------------------ |
| `metrics_addr`    | `[::]:9091`    | HTTP listener.                                   |
| `metrics_port`    | `9091`         | MUST match `metrics_addr`; used by firewall rules. |
| `otlp_endpoint`   | `""`           | Optional OTLP gRPC endpoint.                     |

## common role

Besides packages and the Go toolchain, `common` keeps the root filesystem
bounded on Linux hosts (journald `SystemMaxUse` drop-in plus a
`node-disk-maintenance.timer` that reclaims the apt cache and stale Go build
caches) and carries the opt-in patch path: `ansible-playbook site.yml --tags
os_update` dist-upgrades Debian-family hosts (rebooting when
`/var/run/reboot-required` appears) and runs `freebsd-update` + `pkg upgrade`
on FreeBSD (pending reboots are reported, never performed). Knobs live in
`roles/common/defaults/main.yml`:

| Variable | Default | Effect |
|----------|---------|--------|
| `common_disk_maintenance` | `true` | Install the reclaim timer; `false` removes it |
| `common_disk_maintenance_oncalendar` | `daily` | systemd `OnCalendar` for the timer |
| `common_disk_maintenance_splay_sec` | `3600` | `RandomizedDelaySec` so nodes do not fire in lockstep |
| `common_gocache_max_age_days` | `7` | Go build caches touched within this window are kept |
| `common_journal_max_use` | `300M` | journald `SystemMaxUse` |
| `common_journal_keep_free` | `1G` | journald `SystemKeepFree` |
| `common_journal_max_retention` | `2week` | journald `MaxRetentionSec` |
| `node_exporter_textfile_dir` | `/var/lib/node_exporter/textfile_collector` | Where the reclaim script drops its node_exporter textfile metric |

The reclaim script drops a node_exporter textfile under
`node_exporter_textfile_dir`.

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
manifest_local_binary: "/path/to/local/shard-manifest"
```

When set, the role copies the binary in place of cloning + building. Useful
for air-gapped deployments and during development.
