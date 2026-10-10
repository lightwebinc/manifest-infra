# Ansible

Shared workflow, requirements, variable precedence (set `iface` and `role_hint`
per host), version pins, the `common` role, stat-guarded builds
(`manifest_force_build`) and pre-built binaries (`manifest_local_binary`) are
documented once in [Ansible operations](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/ansible-operations.md). This page
covers what is specific to `shard-manifest`.

The playbook ships three roles applied to the `manifest_nodes` group:

1. `common` — packages + Go toolchain, journald cap + disk-reclaim timer (Linux), opt-in `--tags os_update` patching.
2. `shard-manifest` — daemon install, config render, service unit.
3. `firewall` — perimeter ruleset (Linux nftables, FreeBSD pf).

Run it with:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml
```

## Tags

| Tag         | What it runs                                                  |
| ----------- | ------------------------------------------------------------- |
| `common`    | Base packages + Go toolchain.                                 |
| `manifest`  | Daemon install + config + service.                            |
| `firewall`  | Render and reload the perimeter ruleset.                      |
| `os_update` | Opt-in package upgrade (`--tags os_update`).                  |

## Service variables

Every variable in [`ansible/group_vars/all.yml`](../ansible/group_vars/all.yml),
with its default. `shard_bits`, `mgmt_cidrs_v4` / `mgmt_cidrs_v6` (the play
fails when the firewall is on and both are empty) and `iface` are the ones to
set per deployment.

Generation rollover: leave `manifest_successor_generation_id` empty to disable. When set,
`manifest_successor_transition_epoch` is REQUIRED and must be at least
`now + 2 × announce_interval`. `manifest_successor_shard_bits` must differ from
`shard_bits` by ±1. The successor env vars are only rendered into `config.env`
when `manifest_successor_generation_id` is non-empty.

### shard-manifest source and build

| Variable | Default | Notes |
|---|---|---|
| `go_install_dir` | `/usr/local/go` | Go toolchain install directory |
| `go_version` | pinned in `group_vars/all.yml` | Go toolchain; must be at or above the `go` directive in the service go.mod at the pinned tag |
| `manifest_bin_dir` | `/usr/local/bin` | Binary install directory |
| `manifest_force_build` | `false` | Rebuild even when a binary already exists |
| `manifest_group` | `manifest-infra` | Service group |
| `manifest_install_dir` | `/opt/shard-manifest` | Clone and build directory |
| `manifest_local_binary` | `""` | Pre-built local binary to push; empty = clone and build on the host |
| `manifest_repo` | `https://github.com/lightwebinc/shard-manifest.git` | Git source of the service |
| `manifest_user` | `manifest-infra` | Service user |
| `manifest_version` | pinned in `group_vars/all.yml` | Release tag to build; see [version pins](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/ansible-operations.md#version-pins) |

### Manifest content (BRC-139)

| Variable | Default | Notes |
|---|---|---|
| `authoritative` | `false` | Sets `Flags.Authoritative` on the wire. |
| `generation_id` | `""` | 16-byte hex; bump when `shard_bits` changes. |
| `instance_id` | `""` | OTel service.instance.id; empty = hostname |
| `joined_groups` | `""` | Comma list of indices, or `all`, or empty (identity-only). |
| `manifest_domains` | `""` | DOMAINS: BRC-148 plane descriptors (comma list) id:bits=N[:ssm][:active][:slotspan=S][:generation=HEX32]; empty = none |
| `manifest_encoding` | `auto` | `auto`/`list`/`bitmap`. |
| `role_hint` | `generic` | Informational; one of generic/proxy/listener/retry-endpoint/producer/manifest-only. |
| `shard_bits` | `2` | MUST match the rest of the network. 0..12 per BRC-129. |

### Network

| Variable | Default | Notes |
|---|---|---|
| `iface` | `""` | Egress interface; SHOULD be set per-host to avoid auto-pick surprises. |
| `manifest_port` | `9001` | UDP destination port. |
| `manifest_scope` | `site` | Comma list of `link,site,org,global`. |
| `mc_group_id` | `0x000B` | IANA group-id (BRC-129). |

### Cadence

| Variable | Default | Notes |
|---|---|---|
| `announce_interval` | `300s` | Send period; jittered ±10 % at runtime. |
| `ttl` | `0s` | Wire-format TTL; `0` = consumer default. |

### SSM data-plane advertisement (BRC-139)

| Variable | Default | Notes |
|---|---|---|
| `manifest_publishers` | `""` | Comma list of data-plane publisher IPv6 addresses or DNS names (SSM Sources payload). |
| `manifest_publishers_refresh` | `30s` | DNS re-resolve interval for `manifest_publishers` entries; must be `> 0`. |
| `manifest_source_mode` | `asm` | `asm`/`ssm`. `ssm` REQUIRES `manifest_publishers` to be non-empty. |

### Generation rollover / Successor block (BRC-139 §Successor)

| Variable | Default | Notes |
|---|---|---|
| `manifest_pilot_only` | `false` | Sets `Flags.PilotOnly`: the manifest describes desired fleet state, not the node's own joins (implies `authoritative`). Set on the pilot node driving a rollover. |
| `manifest_successor_generation_id` | `""` | Incoming generation 16-byte hex; empty = no Successor block. |
| `manifest_successor_shard_bits` | `0` | Incoming generation shard bits; must differ from `shard_bits` by ±1. |
| `manifest_successor_source_mode` | `""` | Incoming addressing model `asm`/`ssm`; empty = inherit `manifest_source_mode`. |
| `manifest_successor_transition_epoch` | `0` | Unix seconds at which the successor becomes the sole active generation. |

### Observability

| Variable | Default | Notes |
|---|---|---|
| `log_format` | `json` | text \| json (json default for this daemon) |
| `log_level` | `info` | debug\|info\|warn\|error; runtime-togglable via POST /loglevel + SIGHUP |
| `manifest_debug` | `false` |  |
| `metrics_addr` | `[::]:9091` | HTTP listener. |
| `metrics_port` | `9091` | MUST match `metrics_addr`; used by firewall rules. |
| `otlp_endpoint` | `""` | Optional OTLP gRPC endpoint. |
| `otlp_interval` | `15s` |  |
| `trace_sampling` | `0` | 0..1 trace head sampling (0 = off; exports via otlp_endpoint) |

### Firewall perimeter

| Variable | Default | Notes |
|---|---|---|
| `enable_firewall` | `true` |  |
| `mgmt_cidrs_v4` | `[]` | SSH + Prometheus scrape allow-list. The play fails when the firewall is on and both `mgmt_cidrs_v4` / `mgmt_cidrs_v6` are empty. |
| `mgmt_cidrs_v6` | `[]` | SSH / metrics scrape allow-list (IPv6). |

