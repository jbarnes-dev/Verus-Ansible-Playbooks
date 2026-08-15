# Service-node upgrade options

All added service workflows are opt-in and can be set in inventory variables or
with `--extra-vars`.

For mixed node sets, prefer inventory service groups so the correct options are
selected per host. See `inventory/upgrade-set.example.yml` and
`parallel-upgrade-sets.md` for a parallel lightwallet/explorer/API example.

| Variable | Purpose |
| --- | --- |
| `run_coinsupply` | Runs `verus -rpcwait coinsupply` asynchronously after restart, then waits for it before health checks. |
| `manage_pbaas_chains` | Detects installed candidates from `pbaas_chains`, then stops, restarts, and waits for those chains (`vARRR`, `vDEX`, and `CHIPS` are candidates by default). |
| `check_api_service` | Checks the internal API's known block-1337 hash. |
| `check_insight_service` | Checks that Insight responds and its height is within the configured tolerance of verusd. |
| `check_lightwallet_services` | Checks both lightwalletd and ElectrumX processes and sync heights. |

Typical API node:

```sh
ansible-playbook -i path/to/inventory.yml verus-daemon-upgrade-symbols.yml \
  -e run_coinsupply=true \
  -e manage_pbaas_chains=true \
  -e check_api_service=true
```

Typical Insight explorer:

```sh
ansible-playbook -i path/to/inventory.yml verus-daemon-upgrade-symbols.yml \
  -e run_coinsupply=true \
  -e check_insight_service=true
```

Typical lightwallet node:

```sh
ansible-playbook -i path/to/inventory.yml verus-daemon-upgrade-symbols.yml \
  -e check_lightwallet_services=true
```

The common service checks retry every `service_health_delay` seconds up to
`service_health_retries` times. Heights may differ by
`service_sync_height_tolerance` blocks (default: 2) to avoid failing while a new
block arrives between the two measurements.

While `coinsupply` runs asynchronously, Ansible checks for completion every
`coinsupply_poll_interval` seconds (default: 300, or five minutes).

The Insight and lightwallet checks expect `curl` and `jq`, matching the existing
Zabbix checks. The ElectrumX check also expects `electrumx_rpc`; override
`electrum_rpc_command` if it is installed outside the service user's `PATH`.
