# peakpool

A self-hosted **Stratum V2 solo mining pool** you run with one
`docker compose up`. Each deployment is a full node plus the pool, for **your
own farm**: point your rigs at it and every block you find pays **your** wallet.

- **Solo, not shared.** You earn only when *your* farm finds a block. No
  accounts, no operator taking a cut of your luck, no external pool.
- **Non-custodial by default.** The coinbase pays your wallet directly; the pool
  never holds your coins.
- **Stratum V2 only.** One Noise-encrypted port for every rig.
- **Chain-agnostic core.** The pool is chain-free; each chain is a self-contained
  add-on, so new chains drop in without touching the core.

## Supported chains

| Chain | Deployment |
|-------|-----------|
| ParanO(1)d | [`deployments/parano1d/`](deployments/parano1d/) |

*More chains coming.*

## Quick start

Pick a chain under `deployments/`, then:

```bash
cd deployments/parano1d
cp .env.example .env      # set your image registry + wallet — see that folder's README
docker compose up -d
docker compose logs -f pool
```

Each chain folder has its **own README** with the full walk-through: getting a
wallet, connecting your miners, backing up, and troubleshooting.

## Images

The node and pool run from published container images that `docker compose`
**pulls** automatically — you do not build the pool yourself (its signing
identity and dev fee are baked in at publish time). The exact image tags are
pinned in each deployment's `docker-compose.yaml`:

- `peakminer/parano1d-node:1.0.0`
- `peakminer/peakpool:0.1.0`

## What this is not

- Not a shared/PPS pool — payouts are solo, block-by-block.
- Not a hosting service — you run it on your own machine, for your own rigs.
- Not custodial — the pool moves no money; your wallet is paid on chain.

## License

Proprietary — see [LICENSE](LICENSE). You may run unmodified builds for your own
mining; the disclosed developer fee is the consideration for that right.
