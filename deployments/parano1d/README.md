# parano1d solo mining pool

A ready-to-run **ParanO(1)d full node + Stratum V2 solo pool** in one
`docker compose`. Bring up the stack, point your rigs at it, and every block
your farm finds pays **your** wallet in full. No accounts, no external pool, no
withdrawal — the coinbase is yours.

> **Solo, not shared.** This is a *solo* pool: you only earn when *your* farm
> finds a block. There is no PPS/PPLNS smoothing and no other miners to pool
> luck with. It is meant for one operator running their own rigs.

---

## 1. What you need

- A Linux machine (a VPS or a home box) with **Docker** and the **Docker
  Compose** plugin.
- Outbound internet so the node can follow the chain (P2P port `9600`).
- Nothing else. The node syncs from a snapshot in well under a minute — there is
  no multi-hour initial download.

Check Docker is ready:

```bash
docker version
docker compose version
```

---

## 2. Quick start

From this folder:

```bash
cp .env.example .env      # then EDIT .env — see step 3
docker compose up -d      # pulls images, starts node + pool
docker compose logs -f pool
```

The node syncs first (the pool waits for it automatically). Within a minute or
two you should see the pool banner and a status line every 15 seconds:

```
 ___          _   ___         _
| _ \___ __ _| |_| _ \___ ___| |
|  _/ -_) _` | / /  _/ _ \ _ \ |
|_| \___\__,_|_\_\_| \___/___/_|
# Stratum V2 solo pool · parano1d · v0.1.0

devfee 5%

2026-08-24 16:00:25 ────────────────────────────────────────────────
  uptime 1m35s   height 12122   net-diff 393.5 GH   job 15s   miners 0
  effort 0%   luck —   eta/block —
```

That is the pool running. Now give it a wallet (step 3) and connect your miners
(step 5).

To stop:

```bash
docker compose down          # keeps the chain + wallet volume
```

---

## 3. Set your wallet — where blocks pay

Open `.env` and set **`WALLET_ADDRESS`**. You have two choices.

> 🆕 **Don't have a ParanO(1)d wallet yet?** You don't need to make one first.
> Choose **Option A**: leave `WALLET_ADDRESS` empty and the node **creates a
> fresh wallet for you automatically** on first start (a new `o1…` address plus
> its `wallet.key`). Read the address with the commands in step 4, then back up
> `wallet.key` (step 6) — that address *is* your wallet.

### Option A — use the node's own wallet (easiest, recommended)

Leave it **empty**:

```ini
WALLET_ADDRESS=
```

The node creates and holds a wallet for you automatically on first start, and
blocks pay straight into it. Nothing else to do. To see the address and balance,
use the commands in step 4. (Need more than one address? Generate additional
ones any time with `paranoid_walletNextAddress` — same call style as step 4.)

> ⚠️ The wallet's private key lives in the `chain` Docker volume as
> `wallet.key`. **Back it up** (step 6) — if you lose it, you lose the coins.

### Option B — pay an external wallet you already control

Put your own `o1…` address in:

```ini
WALLET_ADDRESS=o1your0wn0address...............................................
```

Use this if you keep your wallet elsewhere (e.g. a ParanO(1)d wallet on another
machine) and just want the pool to pay it.

> ℹ️ `.env.example` leaves `WALLET_ADDRESS` **empty**, so out of the box blocks
> pay the node's own wallet (Option A). Only set it here if you want an external
> wallet — double-check it is yours.

After changing `.env`, apply it:

```bash
docker compose up -d
```

The pool prints the active payout address on start:

```
farm payout address: o1....
```

Confirm it is yours.

---

## 4. Wallet: address, balance, and sending

These talk to your node over its private RPC (never exposed outside the compose
network). Run them from this folder:

**Your address:**

```bash
docker compose exec node sh -lc '
  curl -fsS -H "Content-Type: application/json" \
    -H "Authorization: Bearer $MINING_KEY" \
    --data "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"paranoid_walletActiveAddress\",\"params\":[]}" \
    http://127.0.0.1:9601'
```

→ `{"result":{"address":"o13n74ug…","key_index":0,"is_active":true}}`

**Your balance:**

```bash
docker compose exec node sh -lc '
  curl -fsS -H "Content-Type: application/json" \
    -H "Authorization: Bearer $MINING_KEY" \
    --data "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"paranoid_walletGetBalance\",\"params\":[]}" \
    http://127.0.0.1:9601'
```

→ `{"result":{"balance_noid":0.0,"spendable_noid":0.0,…}}`

Balances are in **NOID** (`1 NOID = 1,000,000 μNOID`). A freshly mined block is
spendable only after it finalizes (about 18 blocks deep).

**Send NOID:**

```bash
docker compose exec node sh -lc '
  curl -fsS -H "Content-Type: application/json" \
    -H "Authorization: Bearer $MINING_KEY" \
    --data "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"paranoid_walletSend\",\"params\":[\"o1RECIPIENT_ADDRESS\",100000000,0]}" \
    http://127.0.0.1:9601'
```

The three params are `[recipient, amount_μNOID, fee_μNOID]`:

- **recipient** — an `o1…` address to pay.
- **amount** — in **μNOID**: `100000000` = 100 NOID (`1 NOID = 1,000,000 μNOID`).
- **fee** — `0` lets the node pick the fee automatically.

→ `{"result":{"txid":"36b1…","amount_micronoid":100000000,"fee_micronoid":6700,…}}`

The send spends your active address and change returns to it; it confirms once
it lands in a block (a few seconds to a minute). Check `paranoid_walletGetBalance`
again to see it move.

> 🔑 Sending needs the `wallet.key` this node holds. Only this node can spend its
> wallet — **back up `wallet.key` first** (step 6). If you lose it, the coins are
> gone.

---

## 5. Connect your miners

The pool speaks **Stratum V2 only**, on one Noise-encrypted port:
`<pool-host>:34254`.

**peakminer** already has this pool's SV2 identity baked in, so there is nothing
extra to configure — just point each rig at the host:

```bash
peakminer \
    -o stratum+sv2://<pool-host>:34254 \
    -u <your-o1-address>.rig1 \
    --coin parano1d
```

- `-o` — the pool, as `stratum+sv2://host:34254`. On the **same machine** as the
  stack use the Docker bridge address `172.17.0.1`; from **another machine** use
  the pool host's IP or hostname.
- `-u` — this rig's login as `<o1-address>.<worker>`. It must be a valid `o1…`
  address (the pool checks it) and the `.rig1` part just names the worker in the
  status table — give each rig its own name. Blocks pay the wallet you set in
  **step 3**, not this address.
- `--coin parano1d`.

Difficulty auto-tunes to each rig; you do not set it per miner.

Once a rig connects, it shows up in the status table:

```
worker                       hashrate   shares ok/x               diff      last
──────────────────────   ────────────   ───────────────   ────────────   ───────
  rig0                       23.9 MH/s          142 / 1        69.4 MH        2s
──────────────────────   ────────────   ───────────────   ────────────   ───────
TOTAL                       23.9 MH/s          142 / 1
peakpool v0.1.0
```

---

## 6. Back up your wallet — do this once

If you use the node's own wallet (Option A), its private key is the only thing
you cannot recreate. The chain resyncs from the network in seconds; `wallet.key`
does not come back.

```bash
docker compose run --rm backup
```

This copies `wallet.key` into `./backup/`. **Copy `./backup/` off this machine**
(a USB stick, another server, wherever you keep secrets) and keep it safe. Do
this before you accumulate any real balance.

---

## 7. Watch it run

```bash
docker compose logs -f pool          # live banner, share lines, status table
docker compose ps                    # health of node + pool
curl -s http://127.0.0.1:9330/stats  # JSON: hashrate, shares, height, balances
curl -s http://127.0.0.1:9330/healthz
```

The stats endpoint is bound to **loopback only** — it is not reachable from
outside the machine.

What the numbers mean:

- **hashrate** — the *effective* rate the pool measures from accepted shares. It
  dips briefly after each network block while the node proves the next template
  and your rig has no work — a normal few-second pause, see *Troubleshooting*.
- **effort** — work done since the last block vs. the network difficulty
  (100 % ≈ one block's worth of work; solo is spiky, this swings a lot).
- **luck** / **eta/block** — long-run luck and a rough estimate of time to your
  next block at the current hashrate. With one small farm this can be long — that
  is the nature of solo mining.

---

## 8. `.env` reference

| Variable | Meaning | Default |
|---|---|---|
| `WALLET_ADDRESS` | Where blocks pay. Empty = node's own wallet. | *(empty)* |
| `MINING_KEY` | Shared secret between node and pool. Never leaves the compose network. | *(working default)* |
| `NODE_CPUS` | CPU cores for the node. `0` = all. Proving is all-core; give it the machine. | `0` |
| `MIN_SHARE_DIFFICULTY` | Vardiff floor, in expected hashes per share. Rarely needs changing. | `1000000` |

> **`MINING_KEY`:** it authorizes the node's mining RPC. Port `9601` is never
> published to the host, so the default is fine on a machine only you use.
> **Change it** on any shared or multi-tenant box.

---

## 9. Update, restart, reset

```bash
# Update to a newer image release, then recreate:
docker compose pull
docker compose up -d

# Restart just the pool (e.g. after editing .env):
docker compose up -d pool

# Stop everything, keep chain + wallet:
docker compose down

# Wipe EVERYTHING including the wallet (only if you have backed up wallet.key!):
docker compose down -v
```

---

## 10. Troubleshooting

**Pool log says “waiting for the node to sync…” for a while.**
Normal on first start. The node must sync and reach its peer quorum before it
can build templates. `docker compose ps` shows the node as `healthy` once ready;
the pool starts automatically after that.

**`miners 0` in the status table.**
No rig is connected. Check the miner command from step 5 (`-o stratum+sv2://…`,
a valid `o1…` in `-u`, `--coin parano1d`) and that port `34254` is reachable
from the rig (firewall / security group).

**My rig pauses / goes idle for a few seconds between jobs.**
This is normal for ParanO(1)d and not a fault of the pool or your miner.
ParanO(1)d is a *proof-before-nonce* chain: after every new block on the
network, the node must **prove the next template** (a short, all-core step of a
few seconds) before it can hand out fresh work. During that brief window there
is simply no job to grind, so your rig idles until the next template lands, then
resumes immediately. You will see a short gap between `template height N` lines
in the pool log — that is the node proving, working as designed. A faster node
CPU (`NODE_CPUS`) shortens the gap; nothing you do on the miner changes it.

**Occasional `stale` shares.**
Same cause as the pause above. A few shares can land in that proving gap, right
as the tip moves, and get answered `stale`. Small numbers are expected and it
**does not cost you blocks** — a real block is always found on the current
template.

**Blocks paying the wrong address.**
You left the example `WALLET_ADDRESS` in place. Fix `.env` (step 3) and
`docker compose up -d`. Check the `farm payout address:` line on start.

---

## About the dev fee

This build carries a **5 % developer fee**, shown transparently as `devfee 5%`
on startup. It is time-based: for 5 % of mining time the pool builds templates
paying the developer instead of you; the other 95 % pay your wallet. Everything
else about the pool — identity, payout logic — is fixed in the release image.

---

*Powered by peakpool (Stratum V2) · node & pool images `:1.0.0`.*
