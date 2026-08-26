# parano1d solo mining pool

A ready-to-run **ParanO(1)d full node + Stratum V2 solo pool** in one
`docker compose`. Bring up the stack, point your rigs at it, and every block
your farm finds pays **your** wallet in full. No accounts, no external pool, no
withdrawal — the coinbase is yours.

> **Solo, not shared.** This is a *solo* pool: you only earn when *your* farm
> finds a block. There is no PPS/PPLNS smoothing and no other miners to pool
> luck with. It is meant for one operator running their own rigs.

**New here? Follow the steps in order.** Steps 1–3 get the stack running, step 4
gets you a wallet address, step 5 connects your first rig, step 6 backs up the
one file you cannot recreate.

| | |
|---|---|
| [1. What you need](#1-what-you-need) | Docker, ports, hardware |
| [2. Quick start](#2-quick-start) | three commands |
| [3. Set your wallet](#3-set-your-wallet--where-blocks-pay) | where blocks pay |
| [4. Wallet commands](#4-wallet-address-balance-and-sending) | `./p1d address` / `balance` / `send` |
| [5. Connect your miners](#5-connect-your-miners) | the peakminer command line |
| [6. Back up your wallet](#6-back-up-your-wallet--do-this-once) | do this once, early |
| [7. Watch it run](#7-watch-it-run) | logs, stats, what the numbers mean |
| [8. Did I find a block?](#8-did-i-find-a-block) | how to tell, and get paid |
| [9. `.env` reference](#9-env-reference) | every setting |
| [10. Update, restart, reset](#10-update-restart-reset) | day-two operations |
| [11. Troubleshooting](#11-troubleshooting) | when something looks wrong |

---

## 1. What you need

- A **Linux machine** (a VPS or a home box) with **Docker Engine** and the
  **Docker Compose plugin** — install both with the official convenience script
  or your distro packages: <https://docs.docker.com/engine/install/>.
- Your user must be able to talk to Docker. Either add yourself to the `docker`
  group once (`sudo usermod -aG docker $USER`, then log out and back in) or put
  `sudo` in front of every `docker` command below.
- **Hardware:** modest. The chain state is small and resyncs from the network in
  seconds, so there is no multi-hour initial download and no large disk
  requirement. What actually matters is **CPU cores** — the node's proving step
  is all-core and sits on the critical path, so more cores means shorter gaps
  between jobs.
- **Network:** see the port table below.

Check Docker is ready:

```bash
docker version
docker compose version
```

### Ports

| Port | Used by | Who must reach it |
|---|---|---|
| `9600` | Node P2P | The internet, **both directions**. Published on all interfaces; open it in your firewall / cloud security group so peers can connect back. |
| `34254` | **Stratum V2 miners** | Your rigs. If a rig is on another machine, this port must be open to it. |
| `9330` | Pool stats / health | Bound to `127.0.0.1` only — deliberately not reachable from outside the machine. |
| `9601` | Node RPC | Nothing outside. Never published to the host; it stays inside the compose network. |

---

## 2. Quick start

From this folder:

```bash
cp .env.example .env      # then EDIT .env — see step 3
docker compose up -d      # pulls images, starts node + pool
docker compose logs -f pool
```

> **Don't skip the `cp`.** The stack refuses to start without a `.env`, because
> `MINING_KEY` has no built-in default.

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
> its `wallet.key`). Read the address with `./p1d address` (step 4), then back up
> `wallet.key` (step 6) — that address *is* your wallet.

### Option A — use the node's own wallet (easiest, recommended)

Leave it **empty**:

```ini
WALLET_ADDRESS=
```

The node creates and holds a wallet for you automatically on first start, and
blocks pay straight into it. Nothing else to do. To see the address and balance,
use step 4. (Need more than one address? `./p1d newaddress` any time.)

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

`./p1d` is a small script in this folder that talks to your node's private RPC
for you, so you never have to hand-write a `curl` + JSON one-liner. Run it from
this folder. (It pretty-prints with `jq` if you have it — `sudo apt install jq` —
and prints plain JSON otherwise.)

```bash
./p1d help          # all commands
```

**Your address:**

```bash
./p1d address
```

→ `{"result":{"address":"o13n74ug…","key_index":0,"is_active":true}}`

This is the address you use in the miner login in step 5.

**Your balance:**

```bash
./p1d balance
```

→ `{"result":{"balance_noid":0.0,"spendable_noid":0.0,…}}`

Balances are in **NOID** (`1 NOID = 1,000,000 μNOID`). A freshly mined block is
spendable only after it finalizes (about 18 blocks deep).

**Send NOID:**

```bash
./p1d send o1RECIPIENT_ADDRESS 100        # 100 NOID, node picks the fee
```

The script takes the amount in **NOID** (not μNOID), converts it, shows you what
it is about to do, and asks for confirmation before spending. Pass a third
argument to set an explicit fee in μNOID; omit it and the node chooses.

→ `{"result":{"txid":"36b1…","amount_micronoid":100000000,"fee_micronoid":6700,…}}`

The send spends your active address and change returns to it; it confirms once
it lands in a block (a few seconds to a minute). Run `./p1d balance` again to see
it move.

> 🔑 Sending needs the `wallet.key` this node holds. Only this node can spend its
> wallet — **back up `wallet.key` first** (step 6). If you lose it, the coins are
> gone.

<details>
<summary>Prefer raw <code>curl</code>? The equivalent without the script</summary>

```bash
docker compose exec node sh -lc '
  curl -fsS -H "Content-Type: application/json" \
    -H "Authorization: Bearer $MINING_KEY" \
    --data "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"paranoid_walletActiveAddress\",\"params\":[]}" \
    http://127.0.0.1:9601'
```

Swap the `method` for `paranoid_walletGetBalance`, `paranoid_walletNextAddress`,
`paranoid_getNodeStatus`, or `paranoid_walletSend` with
`"params":["o1RECIPIENT",100000000,0]` — that is `[recipient, amount_μNOID,
fee_μNOID]`, and `100000000` μNOID = 100 NOID.

</details>

---

## 5. Connect your miners

The pool speaks **Stratum V2 only**, on one Noise-encrypted port:
`<pool-host>:34254`.

Install **peakminer** on each rig first (see the peakminer release notes for the
build for your hardware), then point each rig at the host. There is nothing else
to configure on the miner side:

```bash
peakminer \
    -o stratum+sv2://<pool-host>:34254 \
    -u <your-o1-address>.rig1 \
    --coin parano1d
```

- `-o` — the pool, as `stratum+sv2://host:34254`. If peakminer runs on the
  **same machine** as this stack, use `127.0.0.1:34254`. From **another
  machine**, use the pool host's LAN IP or hostname. (Only if you run peakminer
  inside *another container* do you need the Docker bridge address, typically
  `172.17.0.1`.)
- `-u` — this rig's login as `<o1-address>.<worker>`. Put **any valid `o1…`
  address** here — **it does NOT affect where coins go.** Every block pays the
  wallet you set in **step 3**, whatever you write here; this field is only a
  worker label for the status table. The pool does check the format, so it must
  be a real `o1…` address (a malformed one is rejected) — simplest is to reuse
  your own address from **step 4** (`./p1d address`). The `.rig1` suffix just
  names the worker, so give each rig its own.
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

If you use the node's own wallet (Option A), its key is the only thing you
cannot recreate. The chain resyncs in seconds; the wallet does not.

```bash
./p1d backup                 # same as: docker compose run --rm backup
```

This copies the **full wallet set** — `wallet.key` **plus** `wallet.meta` and
`.network-storage-epoch` — into `./backup/`. **Copy `./backup/` off this
machine** (a USB stick, another server, wherever you keep secrets), before you
accumulate any real balance.

> ⚠️ All three files matter. On a fresh machine the node runs a one-time reset
> that **wipes a data directory that is missing `.network-storage-epoch`** and
> then creates a new, empty wallet — so a lone `wallet.key` does **not** restore.

**To restore** (e.g. on a new machine): bring the stack up once, put your backup
files into `./backup/`, then run one command:

```bash
docker compose up -d          # first run creates the volume + marker
# copy your backed-up wallet.key, wallet.meta and .network-storage-epoch into ./backup/
./p1d restore                 # stops the node, restores the wallet, starts it
```

`./p1d restore` copies the **full** wallet set (including `.network-storage-epoch`)
back into the chain volume. The node log should then say **`loading wallet`** (not
`creating new wallet`); verify with `./p1d address` and `./p1d balance`.

> Docker creates `./backup/` owned by **root**, so copying it out may need
> `sudo`. Treat these files like private keys — because they are.

---

## 7. Watch it run

```bash
./p1d logs                           # live banner, share lines, status table
./p1d stats                          # JSON: hashrate, shares, height, balances
./p1d status                         # node: sync height, mining_ready
docker compose ps                    # health of node + pool
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

## 8. Did I find a block?

Solo mining is all-or-nothing, so this is the question you will actually ask.
Three ways to check, cheapest first:

```bash
docker compose logs pool | grep -i block     # every block the pool found
./p1d balance                                # the money side
./p1d stats                                  # blocks + effort since last one
```

- The pool logs a line when one of your shares turns out to be a **block**, and
  the **effort** counter in the status table resets to ~0 % right after — a
  reset effort with no restart is itself a good sign you just found one.
- The reward lands in the payout address from step 3. It shows up in
  `./p1d balance` as balance first, and becomes **spendable only after the block
  finalizes** (about 18 blocks deep), so a fresh block reads as balance without
  spendable for a few minutes. That is normal, not a stuck payment.
- If you chose **Option B** (an external wallet), the node here holds none of it
  — check the balance in *that* wallet instead; `./p1d balance` will stay at
  zero forever, correctly.

---

## 9. `.env` reference

| Variable | Meaning | `.env.example` value |
|---|---|---|
| `WALLET_ADDRESS` | Where blocks pay. Empty = node's own wallet. | *(empty)* |
| `MINING_KEY` | Shared secret between node and pool. **Required** — no built-in default; the two services read this same value, so any non-empty string matches. Never leaves the compose network. | `change-me-to-any-random-secret` |
| `NODE_CPUS` | CPU cores for the node. `0` = all. Proving is all-core; give it the machine. | `0` |
| `MIN_SHARE_DIFFICULTY` | Vardiff floor, in expected hashes per share. Rarely needs changing. | `1000000` |

> **`MINING_KEY`:** it authorizes the node's mining RPC. Port `9601` is never
> published to the host, so nothing outside the compose network can use it — but
> the value shipped in `.env.example` is a placeholder, not a secret. **Set your
> own**, especially on a shared or multi-tenant box:
>
> ```bash
> openssl rand -hex 32
> ```
>
> Changing it later is fine — edit `.env` and `docker compose up -d`; both
> services pick up the new value together.

---

## 10. Update, restart, reset

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

## 11. Troubleshooting

**`permission denied … /var/run/docker.sock`**
Your user is not in the `docker` group. Either `sudo usermod -aG docker $USER`
and log out and back in, or prefix every command with `sudo` (including
`./p1d`, e.g. `sudo ./p1d address`).

**`error while interpolating … MINING_KEY: set MINING_KEY in .env`**
You have no `.env`. Run `cp .env.example .env` in this folder first — there is
no built-in default for `MINING_KEY`.

**`bind: address already in use` on `34254`, `9600`, or `9330`.**
Something else already listens there — often an older copy of this stack. Check
with `docker compose ps` and `sudo ss -lptn 'sport = :34254'`, then stop the
other process (or `docker compose down` the old stack) and try again.

**`pull access denied` / `manifest unknown` when starting.**
The images could not be fetched. Check the machine has outbound internet and
that the tags in `docker-compose.yaml` match a published release; if the images
are private for you, `docker login` first.

**Pool log says "waiting for the node to sync…" for a while.**
Normal on first start. The node must sync and reach its peer quorum before it
can build templates. `docker compose ps` shows the node as `healthy` once ready;
the pool starts automatically after that. `./p1d status` shows the same thing as
`mining_ready`.

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

*Powered by peakpool (Stratum V2) · images `peakminer/parano1d-node:1.0.0` and
`peakminer/peakpool:0.1.1`.*
