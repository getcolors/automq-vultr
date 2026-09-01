# automq-vultr

Desired state for a three-node [AutoMQ](https://github.com/AutoMQ/automq)
cluster on Vultr, run by the [`automq`](https://github.com/getcolors/automq)
Package Skill. Kafka 3.9.1 wire protocol, storage in Cloudflare R2, a public
`SASL_SSL` endpoint with SCRAM authentication and ACL authorization.

| | |
|---|---|
| Bootstrap | `automq.bigconfig.online:9092` |
| Brokers | `b0` / `b1` / `b2``.automq.bigconfig.online:9092` |
| Compute | 3 × `vc2-4c-8gb`, `ams`, one VPC |
| Open ports | 22 and 9092 — the quorum (9093) and inter-broker traffic (9094) stay on the VPC |
| Storage | R2 buckets `automq-data` and `automq-ops` |
| Cost | ~$120/month |

## Connecting

```sh
kcat -b automq.bigconfig.online:9092 \
  -X security.protocol=SASL_SSL -X sasl.mechanism=SCRAM-SHA-512 \
  -X sasl.username=automq -X sasl.password="$(ssh automq-vultr sudo automq-credential | sed -n 's/^password: *//p')" \
  -L
```

The client principal may produce and consume under the `colors-` prefix and
nothing else. It is not a superuser: 9092 faces the internet, and
authentication alone is not a boundary.

## Replication factor 1

Deliberate. AutoMQ acknowledges a produce once the record is in R2, so replicas
would add cost and write amplification without adding durability — upstream
ships RF=1 for the same reason. The three nodes exist for the controller
quorum, partition failover and throughput.

What that does not buy is availability: a partition whose leader dies is
unwritable until it is reassigned. The acceptance run measures that window
against a partition chosen *because* the killed broker leads it.

## Editing

`colors.yml` is the only file to edit; it holds non-secret values only.
`./green build` and `./green create --dry-run` are credential-free and are the
safe way to check a change.

See [CLAUDE.md](CLAUDE.md) for operating, deleting, and the launcher-copy trap.
