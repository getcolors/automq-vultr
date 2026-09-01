# CLAUDE.md

## Repository

`automq-vultr` is a deployment: desired state only, no source code. It runs the
`automq` Package Skill — a three-node AutoMQ 1.7.4 cluster (Kafka 3.9.1 wire
protocol) on Vultr, with Cloudflare R2 as the storage tier.

`colors.yml` is the only file to edit. Everything else is either generated
(`.colors/`), secret (`.envrc.private`), or an installed copy of the package
(`.agents/skills/package-automq-green/`, and the root `./green` launcher).

## Shape

Three `vc2-4c-8gb` instances in `ams`, one Vultr VPC, one firewall opening
**22 and 9092 only**. Cloudflare `bigconfig.online` holds three A records on
`automq.bigconfig.online` and one per broker (`b0`/`b1`/`b2`), all DNS-only.
Two R2 buckets, `automq-data` and `automq-ops`, adopted rather than created.

**Replication factor is 1 and that is the architecture.** Durability is R2, not
replicas. The three nodes buy the controller quorum, partition failover and
throughput. Do not "fix" it.

## Cost

Three nodes at $40/month each, so **$120/month** while this deployment exists,
plus R2 storage and operations. It is deliberately left running.

## Credentials

`.envrc.private`, gitignored, never read or echoed:
`COLORS_PAR_VULTR_API_KEY`, `COLORS_PAR_CLOUDFLARE_API_TOKEN`,
`COLORS_PAR_R2_ACCESS_KEY_ID` / `_SECRET_ACCESS_KEY` (state backend), and
`COLORS_PAR_AUTOMQ_R2_ACCESS_KEY_ID` / `_SECRET_ACCESS_KEY` (the storage
token, scoped to the two AutoMQ buckets — the only credential that reaches the
hosts).

Every password inside the cluster is generated on node 0 at first converge:
the four SASL principals, their SCRAM salts, and the keystore password. None
is operator-supplied. Never export `COLORS_PAR_PROFILE`.

## Commands

```sh
direnv allow          # once, after the toolchain is installed
./green build         # render .colors/ — credential-free
./green validate      # desired state, tools, Vultr access
./green create --dry-run
./green create        # converge for real
```

Operating the cluster, over `ssh automq-vultr` (node 0) or `ssh automq-vultr-<n>`:

```sh
automq-status         # quorum, brokers, partitions, certificate expiry
sudo automq-credential  # root only: the client SASL credential
sudo automq-smoke     # re-run the on-host gates
```

## Two firewalls, not one

These nodes sit behind the Vultr firewall group (desired state, in the compute
stage) **and** ufw, which the Vultr Ubuntu image ships enabled with a single
`22/tcp` rule. The converge opens the cluster ports in both. This matters when
diagnosing: ufw passes ICMP, so every node can ping every other node while
every inter-node TCP connection is dropped — the quorum never elects and
nothing in Kafka's output mentions a firewall.

Test raw TCP across the VPC, never ping, and check `ufw status` on the host
before the provider's console.

## Never run `build` during a converge

`.colors/` is live input to the running stage, not a scratch copy.
Re-rendering it under a running script makes bash resume mid-token and report
a syntax error in a file that is perfectly valid.

## Deleting

`compute-prevent-destroy: true` is committed and must stay committed. A destroy
needs a deliberate one-run override:

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

**`delete` does not empty the buckets.** They hold the cluster's data, and a
teardown that erased them would make an accidental delete unrecoverable.
Purging is separate and manual — including the `_colors/automq-vultr/` markers,
without which the buckets cannot be adopted again.

## The launcher is a copy

The root `./green` is a **copy** of
`.agents/skills/package-automq-green/green`, not a symlink. `npx skills update
-p` rewrites the payload and leaves the root file alone, so always:

```sh
npx skills update -p
cp .agents/skills/package-automq-green/green green
```

Otherwise this deployment keeps running the old pin while `skills-lock.json`
claims the new one.

## Git

Work on the current branch. Do not commit or push unless explicitly authorized.
