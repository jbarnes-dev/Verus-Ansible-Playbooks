# Verus daemon upgrade playbook

This repository contains an Ansible playbook for upgrading Verus (`verusd` and
`verus`) on one or more Linux nodes with symbol-enabled builds from
`symbuild.verus.io`.

The playbook can discover the latest public Verus release from GitHub or install
an explicitly selected, unpublished version. It can also wait for a node to be
removed from public DNS, manage PBaaS daemons, run `coinsupply`, and verify API,
Insight, lightwalletd, and ElectrumX services after the upgrade.

## What the playbook does

For every selected host, `verus-daemon-upgrade-symbols.yml`:

1. Reads the installed `verusd` version.
2. Selects either the latest GitHub release or a manually supplied version.
3. Ends work on hosts that do not need the selected version.
4. Optionally waits until the host's public IP is absent from all configured
   service DNS records.
5. Downloads the corresponding symbol build from `symbuild.verus.io`.
6. Optionally stops the configured PBaaS chains.
7. Stops `verusd`, replaces `verusd` and `verus`, and restarts the daemon.
8. Optionally starts `coinsupply` while the daemon and PBaaS RPCs recover.
9. Runs the health checks enabled for that host.

The play uses Ansible's `free` strategy. Hosts advance independently, while
`--forks` sets the maximum number of hosts Ansible operates on concurrently.

## Requirements

### Control machine

- Ansible Core with the `ansible.builtin` modules used by this repository.
- SSH access to every managed host.
- Access to `https://dns.google/resolve` when the DNS safety check is enabled.
- A populated inventory whose target hosts belong to the `verus_nodes` group.

### Managed nodes

- A Linux x86-64 host with Python available for Ansible.
- An SSH account that can use privilege escalation (`become`).
- A Verus installation owned by the `verus` user. The current extraction and
  download tasks assume the user and home directory are `verus` and
  `/home/verus`, even though most other paths are configurable.
- Outbound access to the GitHub API when automatic release discovery is used,
  and to `https://symbuild.verus.io` for the archive download.
- `bash`, `tar`, and `pgrep`.
- `curl` for the Insight check, and `jq` for the lightwallet checks.
- `electrumx_rpc` in the Verus user's `PATH` when the ElectrumX check is enabled,
  unless `electrum_rpc_command` is overridden.

By default, the Verus binaries and data directory are expected at:

```text
/home/verus/bin/verusd
/home/verus/bin/verus
/home/verus/.komodo/VRSC
```

## Quick start

1. Copy the example inventory and replace its documentation-only addresses:

   ```sh
   cp inventory/upgrade-set.example.yml inventory/upgrade-set-01.yml
   ```

2. Add one public service hostname or URL per line to `service-dns.md`. Leave
   the file with no endpoint entries if the selected nodes do not serve public
   DNS names.

3. Confirm inventory connectivity and privilege escalation:

   ```sh
   ansible -i inventory/upgrade-set-01.yml verus_nodes -m ping --become
   ```

4. Run the upgrade, setting `--forks` to at least the number of hosts when the
   entire set should operate concurrently:

   ```sh
   ansible-playbook \
     -i inventory/upgrade-set-01.yml \
     --forks 3 \
     verus-daemon-upgrade-symbols.yml
   ```

The default behavior discovers the latest release from
`veruscoin/veruscoin`. A host is skipped when that release is not newer than
the installed version.

## Inventory and service types

The example inventory organizes hosts by service role and enables only the
workflows each role needs:

```yaml
verus_nodes:
  children:
    lightwallet_nodes:
      hosts:
        lightwallet-01:
          ansible_host: 192.0.2.11
      vars:
        check_lightwallet_services: true

    explorer_nodes:
      hosts:
        explorer-01:
          ansible_host: 192.0.2.12
      vars:
        run_coinsupply: true
        check_insight_service: true

    api_nodes:
      hosts:
        api-01:
          ansible_host: 192.0.2.13
          # Use the public service address if ansible_host is private.
          service_dns_target_ip: 198.51.100.13
      vars:
        run_coinsupply: true
        manage_pbaas_chains: true
        check_api_service: true
```

Inventory group variables and host variables override the defaults in
`roles/verus_upgrade_defaults/defaults/main.yml`. The same variables may also
be supplied with `--extra-vars`, but inventory groups are clearer for mixed
service sets.

To use a larger shared inventory, put the manually selected nodes in a group
and limit the play to it:

```sh
ansible-playbook \
  -i inventory/production.yml \
  --limit upgrade_set_01 \
  --forks 3 \
  verus-daemon-upgrade-symbols.yml
```

Ensure every limited host is also a member of `verus_nodes`.

## Selecting a version

### Latest public GitHub release

This is the default:

```sh
ansible-playbook -i inventory/upgrade-set-01.yml \
  verus-daemon-upgrade-symbols.yml
```

With `git: true`, the target is GitHub's latest release tag and an upgrade is
performed only if that tag is newer than the installed version.

### Explicit symbols build

For a version that exists on the symbols server but is not yet a public GitHub
release, disable discovery and provide its exact artifact version:

```sh
ansible-playbook -i inventory/upgrade-set-01.yml \
  -e git=false \
  -e verus_version=v1.2.3 \
  verus-daemon-upgrade-symbols.yml
```

In manual mode, the play upgrades whenever the requested version differs from
the installed version. Confirm that this archive exists before starting:

```text
https://symbuild.verus.io/Verus-CLI-Linux-VERSION-x86_64-WithSymbols.tar.gz
```

## DNS safety check

The DNS check is enabled by default. Before stopping a daemon, each host waits
until its `service_dns_target_ip` is no longer present in the public DNS answers
for any endpoint listed in `service-dns.md`.

The endpoint file accepts a hostname or URL on each non-comment line. Markdown
bullets, URL schemes, ports, paths, blank lines, headings, and HTML comments
are handled. For example:

```md
# Public service endpoints

- https://api.example.com/status
- explorer.example.com:443
lightwallet.example.com
```

Queries use Google Public DNS's JSON API and select `A` or `AAAA` based on the
target address. A resolver error is treated as unsafe. The default behavior is
to retry every five minutes for up to 60 attempts, then fail the host without
performing its upgrade.

`service_dns_target_ip` defaults to `ansible_host`. Override it per host if SSH
uses a private address or hostname while public services use another IP:

```yaml
api-01:
  ansible_host: 10.0.0.13
  service_dns_target_ip: 203.0.113.13
```

To intentionally disable the guard for a host or group:

```yaml
service_dns_check: false
```

## Optional service workflows

All service-specific workflows are disabled by default.

| Variable | Effect |
| --- | --- |
| `run_coinsupply` | Starts `verus -rpcwait coinsupply` asynchronously after restart and waits for successful completion before health checks. |
| `manage_pbaas_chains` | Detects installed candidates from `pbaas_chains`, then stops, restarts, and waits for those chains. |
| `check_api_service` | Calls the internal JSON-RPC API and validates the configured known block hash. |
| `check_insight_service` | Compares Insight's block height with the local daemon height. |
| `check_lightwallet_services` | Checks lightwalletd and ElectrumX processes and their sync heights. |

The default PBaaS chain candidates are `vARRR`, `vDEX`, and `CHIPS`. On each
host, the play resolves their identifiers and manages only candidates with a
regular configuration file at
`pbaas_data_dir/CHAIN_ID/CHAIN_ID.conf`. Missing candidates are reported and
skipped, so hosts do not need to run the same set of chains. The API health
check expects the block at height 1337 to match the configured VRSC hash.
Insight, lightwalletd, and ElectrumX are considered synchronized when their
height is within two blocks of the associated daemon. All of these settings are
overridable.

Common service examples are available in `service-node-options.md`.

## Important variables

The complete defaults are in `roles/verus_upgrade_defaults/defaults/main.yml`.
Frequently changed values include:

| Variable | Default | Description |
| --- | --- | --- |
| `verus_user` | `verus` | User that runs Verus commands. See the current fixed-path limitation in Requirements. |
| `verus_home` | `/home/verus` | Working directory used during extraction and startup. |
| `verus_executable_dir` | `/home/verus/bin` | Location of `verusd` and `verus`. |
| `verus_data_dir` | `/home/verus/.komodo/VRSC` | VRSC data directory. |
| `git` | `true` | Discover the latest GitHub release. |
| `verus_version` | empty | Required target version when `git=false`. |
| `ext_base_url` | `https://symbuild.verus.io` | Base URL for symbol builds. |
| `service_dns_check` | `true` | Require public DNS to be clear before upgrading. |
| `service_dns_file` | `service-dns.md` | Controller-side endpoint list. |
| `service_dns_retry_delay` | `300` | Seconds between DNS queries. |
| `service_dns_max_checks` | `60` | Maximum DNS attempts per host. |
| `pbaas_chains` | `vARRR`, `vDEX`, `CHIPS` | Candidate PBaaS daemons; only locally installed candidates are managed. |
| `coinsupply_async_timeout` | `14400` | Maximum seconds allowed for `coinsupply`. |
| `coinsupply_poll_interval` | `300` | Seconds between asynchronous `coinsupply` completion checks. |
| `service_health_retries` | `60` | Maximum attempts for each service health operation. |
| `service_health_delay` | `10` | Seconds between service health attempts. |
| `service_sync_height_tolerance` | `2` | Accepted block-height difference. |

## Failure and rerun behavior

- A failure is isolated to its host because the play uses `strategy: free`;
  other selected hosts continue independently.
- If a host is already on the latest discovered GitHub release, the play ends
  for that host before DNS checks and downloads.
- A failed DNS check leaves the daemon untouched.
- Failures after the daemon has stopped may require manual recovery. Inspect
  the failed task and the node's daemon logs before rerunning.
- Rerunning with a manual version equal to the installed version skips that
  host. If binaries need to be replaced with the same version, this playbook
  does not currently expose a force-reinstall option.

## Repository layout

| Path | Purpose |
| --- | --- |
| `verus-daemon-upgrade-symbols.yml` | Main upgrade playbook. |
| `roles/verus_upgrade_defaults/` | Low-precedence defaults for inventory and CLI overrides. |
| `inventory/upgrade-set.example.yml` | Example mixed-service upgrade set. |
| `service-dns.md` | Public endpoint list read by the DNS guard. |
| `service-dns-check.yml` | DNS query and retry task sequence. |
| `pbaas-stop.yml`, `pbaas-start.yml` | Optional PBaaS lifecycle tasks. |
| `coinsupply-start.yml`, `coinsupply-wait.yml` | Optional asynchronous `coinsupply` tasks. |
| `health-*.yml` | Optional service-specific recovery checks. |
| `parallel-upgrade-sets.md` | More detail on parallel sets and DNS handoff. |
| `service-node-options.md` | Service-role examples and health-check notes. |

## Suggested operating sequence

For rolling service maintenance:

1. Select a small upgrade set and remove or move its service records.
2. Wait for the playbook's public DNS guard to confirm the old IP is gone.
3. Let the playbook upgrade the set and complete every enabled health check.
4. Restore the set to service according to your normal traffic-management
   process.
5. Repeat with the next set.

See `parallel-upgrade-sets.md` for additional examples.
