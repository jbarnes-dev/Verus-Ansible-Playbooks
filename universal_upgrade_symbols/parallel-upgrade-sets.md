# Parallel upgrade sets

The play targets the `verus_nodes` inventory group and uses Ansible's `free`
strategy. Every host in the selected set can therefore move through DNS
waiting, daemon replacement, recovery, and health checking independently.
One slow node does not hold the others at the same task.

## One inventory file per set

Copy `inventory/upgrade-set.example.yml` for the set being drained from DNS.
The example contains one lightwallet, one explorer, and one API node, with each
service workflow enabled through inventory group variables.

After replacing the example names and addresses, run a three-node set with:

```sh
ansible-playbook \
  -i inventory/upgrade-set-01.yml \
  --forks 3 \
  verus-daemon-upgrade-symbols.yml
```

For an unpublished symbols build, add the existing manual-version overrides:

```sh
ansible-playbook \
  -i inventory/upgrade-set-01.yml \
  --forks 3 \
  -e git=false \
  -e verus_version=v1.2.3 \
  verus-daemon-upgrade-symbols.yml
```

`--forks` is the maximum number of hosts Ansible operates on concurrently. Set
it to at least the number of nodes in the set for full parallelism. Ansible's
default is five, but making it explicit documents the intended set size.

## Using a shared inventory and `--limit`

The same play can use a normal production inventory if its nodes are members of
`verus_nodes`. Put the manually chosen nodes in a set group such as
`upgrade_set_01`, then select only that intersection:

```sh
ansible-playbook \
  -i inventory/production.yml \
  --limit upgrade_set_01 \
  --forks 3 \
  verus-daemon-upgrade-symbols.yml
```

## DNS handoff between sets

For each set, remove or move its service records, then launch the play. Each
host independently checks every endpoint in `service-dns.md` for its own
`service_dns_target_ip` and waits until that address has disappeared. Once the
set finishes and passes its service-specific health checks, make the next DNS
change and invoke the next inventory file or limit group.

If SSH uses a private address while public DNS uses another address, set
`service_dns_target_ip` on that host in inventory. All defaults use role-default
precedence, so child-group and host variables in inventory can override them.
