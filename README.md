# miner

Single-binary Monero (RandomX) miner for any Mac — Apple Silicon or Intel.

- **One file**, 7.4 MB. Universal binary; runs on every Mac released in the
  last decade.
- **Three front-ends from the same binary**: plain-text status (default),
  full terminal UI, or a native macOS window.
- **Embedded web dashboard** at `http://localhost:9090` whenever the miner
  is running.
- **Pool / P2Pool / Solo modes.** Pool is default; the others are one flag away.

## Install

Grab the latest binary from this repo's [**Releases** page](../../releases/latest).

Easiest, from a Mac terminal:

```bash
# 1. Download the latest release directly
curl -L -o miner https://github.com/USER/REPO/releases/latest/download/miner

# 2. Drop the Gatekeeper quarantine attribute that browsers / curl set
xattr -d com.apple.quarantine miner 2>/dev/null || true

# 3. Mark it executable
chmod +x miner

# 4. Optionally move it onto your PATH
mv miner /usr/local/bin/

# 5. Verify
miner info
```

If `miner info` prints a hardware summary (CPU, cores, RAM), you're set.
Replace `USER/REPO` in the URL above with this repo's actual path.

## Quick start

You need a Monero wallet address to mine. Generate one with
`monero-wallet-cli --offline` (`brew install monero` first) or with any
XMR wallet (Cake / Feather / Monero GUI). The **address only** — never
paste your 25-word seed anywhere.

```bash
# Pool mining, plain text output (default)
miner mine --wallet 4YOUR_XMR_ADDRESS

# Same thing, with the full terminal UI
miner mine --tui --wallet 4YOUR_XMR_ADDRESS

# Same thing, in a native macOS window
miner mine --gui --wallet 4YOUR_XMR_ADDRESS
```

All three modes show the same live data. The web dashboard is also live
at `http://127.0.0.1:9090/` regardless of mode.

`Ctrl-C` quits (any mode). In `--tui` mode you can also press `q`.

## Front-end modes

| Mode | How | Notes |
|---|---|---|
| **Plain text** (default) | no flag | One status line per second to stderr. Survives `nohup` / `tmux` / supervisors. |
| **Terminal UI** | `--tui` | Full ratatui dashboard in the current terminal. Minimum 72×24. |
| **Native window** | `--gui` | macOS `NSWindow` + WKWebView wrapping the dashboard. Real Mac chrome. Title shows the detected hardware. |

The web page at `http://127.0.0.1:9090/` is always available too — view
it in any browser. Pass `--web-bind 0.0.0.0:9090` to access from another
device on your LAN.

## All flags

| Flag | Default | Meaning |
|---|---|---|
| `--wallet <ADDR>` | required (except `--mock`) | Your 95-char XMR address. Rewards arrive here. |
| `--pool <HOST:PORT>` | `pool.supportxmr.com:3333` | Stratum endpoint. Any Monero pool works. |
| `--password <STR>` | `x` | Pool worker password. Most pools accept anything. |
| `--workers <SET>` | `auto` | `auto` / `p-cores` / `e-cores` / `all`. `auto` picks based on chassis. |
| `--tui` | off | Terminal UI in this terminal (mutually exclusive with `--gui`). |
| `--gui` | off | Native macOS window. |
| `--web-bind <ADDR\|off>` | `127.0.0.1:9090` | Embedded dashboard server. `0.0.0.0:9090` for LAN access. `off` disables. |
| `--solo` | off | Solo mining via monerod RPC. See "Solo mining" below. |
| `--solo-node <URL>` | a public node | RPC endpoint for `--solo`. Point at `http://127.0.0.1:18081` if you run your own monerod. |
| `--p2pool` | off | Mine to a locally-running P2Pool daemon. |
| `--p2pool-addr <HOST:PORT>` | `127.0.0.1:3333` | Override P2Pool stratum endpoint. |
| `--mock` | off | Synthetic data — no real pool, no real hashing. For trying the UI. |
| `--stderr-log <PATH\|auto\|off>` | `auto` | Where TUI mode redirects stderr (so library prints don't corrupt the screen). |

Subcommands beyond `mine`:

- `miner info` — print detected hardware + recommended config, exit
- `miner selftest` — validate the RandomX hasher against canonical test vectors
- `miner stratum-test --wallet <ADDR>` — verify the stratum protocol against a pool, exit

`miner mine --help` and `miner --help` print the canonical reference.

## Dev fee

This binary contributes **1% of its mining time** (1 minute per 100) to
the maintainer's wallet. Every 99 minutes of mining for you is followed
by 1 minute of mining for `4BBS…te1`. This is how the binary stays
maintained as a free download.

It's disclosed in the events log on startup:

```
· dev fee: 1.0% (99min user / 1min dev per cycle)
```

And every time the dev window opens / closes:

```
· ▶ dev fee window: 60s
· ◀ dev fee window closed
```

The fee is **only applied in pool and P2Pool modes**. Solo mode is
exempt — block variance makes time-slicing meaningless there.

If 1% isn't acceptable for your use case, this binary isn't the right
fit. Use a miner with a configurable fee (XMRig allows 0%). The
percentage is hardcoded by design.

## Solo mining

```bash
# Connect to a community public node (default — easy but trusts a third party)
miner mine --solo --wallet 4YOUR_XMR_ADDRESS

# Or point at your own monerod
miner mine --solo --solo-node http://127.0.0.1:18081 --wallet 4YOUR_XMR_ADDRESS
```

Solo mines against network difficulty (currently ~800 G). At consumer
hashrates this is a multi-year lottery — but the prize (~0.65-0.9 XMR
per block) goes entirely to your wallet.

## P2Pool mining

P2Pool is a decentralized sidechain pool — payouts come straight from
the Monero block coinbase, no operator holds your XMR.

You need `p2pool` and `monerod` running locally first. Install p2pool from
the upstream releases page (Homebrew doesn't ship a formula). Run monerod
with `--zmq-pub tcp://127.0.0.1:18083`, then run p2pool against it
(`p2pool --wallet 4YOUR... --mini`). Once p2pool's stratum is up on
`127.0.0.1:3333`, point this miner at it:

```bash
miner mine --p2pool --wallet 4YOUR_XMR_ADDRESS
```

The `--mini` flag on p2pool selects the low-hashrate sidechain — right
pick for a CPU mining fleet (1-50 kH/s combined).

## Running detached on a remote Mac (tmux)

```bash
ssh user@remote-mac '
  tmux new-session -d -s miner "caffeinate -i ~/miner mine \
    --wallet 4YOUR_XMR_ADDRESS \
    --web-bind 0.0.0.0:9090"
'

# Check on it later
ssh user@remote-mac 'tail -f ~/.tmux-output'   # or attach: ssh -t user@remote-mac 'tmux attach -t miner'

# Stop
ssh user@remote-mac '
  tmux send-keys -t miner C-c
  sleep 1
  tmux kill-session -t miner 2>/dev/null
'
```

`caffeinate -i` keeps the Mac awake while mining. Without it the Mac
sleeps and mining stops.

## Hashrate reference

Approximate RandomX hashrates in dataset mode, P-cores only:

| Mac | Workers | ~Hashrate |
|---|---|---|
| Mac mini (M1) | 4 | 1.8 kH/s |
| MacBook Pro (M1 Max) | 8 | 3.2 kH/s |
| MacBook Pro (M3 Max) | 12 | ~4.5 kH/s (extrapolated) |
| Mac Studio (M2 Ultra) | 16 | ~6 kH/s (extrapolated) |

Roughly 85% of XMRig's published numbers — the gap is in the RandomX
inner loop, not the scheduler.

## Troubleshooting

### macOS won't open it ("cannot be opened because…")

The download set the Gatekeeper quarantine attribute. Remove it:

```bash
xattr -d com.apple.quarantine /path/to/miner
```

Or right-click the binary in Finder → Open → "Open Anyway".

### It crashes on first hash (SIGBUS)

This binary is codesigned with the JIT entitlement that macOS Apple
Silicon requires for RandomX. If you've modified or copy-edited the
file, that signature is invalidated and the JIT pages get rejected.
Re-download a fresh, unmodified copy.

### Pool connection refused / timeout

Confirm the `--pool` host and port are right; some pools also offer
TLS-only stratum on different ports (this miner doesn't speak stratum-TLS).
Try a different pool or check the pool's status page.

### Hashrate seems low

Other processes (especially `mediaanalysisd`, Spotlight indexing,
Photos.app) compete for CPU. Check with `top -o cpu`. Kill noisy
neighbors with `sudo killall <process>` or wait for them to finish.

## Privacy notes

- **Pool mode**: the pool sees your IP and the wallet address you're
  mining to. SupportXMR's stats page makes wallet addresses publicly
  searchable.
- **Solo mode with public node**: the node operator can correlate IP →
  wallet. Run your own monerod for privacy.
- **P2Pool**: peers on the P2Pool network see your share submissions.
  This is fundamental to decentralization.
- **The miner itself** does not phone home or report any telemetry. The
  embedded web server binds to `127.0.0.1` by default — local-only.

## License

Binary distribution. The RandomX hashing library (statically linked
inside this binary) is BSD-3 licensed —
see <https://github.com/tevador/RandomX>.

## Issues / contact

File an issue in this repo with: macOS version, Mac model (`miner info`
output), the command you ran, and any error message. The events log
(in the TUI / web UI / `--no-tui` stderr) usually has what's wrong.
