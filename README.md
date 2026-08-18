# Raiku Shredstream Proxy

A forward-only UDP shred relay. It listens on a bound port, de-duplicates what
arrives, and fans each packet out to one or more destinations — typically a
local validator's TVU port.

Forked from [jito-labs/shredstream-proxy](https://github.com/jito-labs/shredstream-proxy)
at commit `b96e369` (tag `v0.2.14`) and trimmed to the forwarding path only. See
[`NOTICE`](NOTICE) for the full list of modifications. Licensed under Apache 2.0.

## Why this fork exists

Jito announced that ShredStream is deprecated and shuts down on September 5, 2026.
Upstream `b96e369` added a hard-coded date check that aborts the `shredstream`
subcommand after that date, and upstream is winding down — including the
`jito-labs/mev-protos` submodule and the `jito-foundation/jito-solana` branch that
the build depended on.

Raiku only ever used the `forward-only` path, which the upstream kill switch does
not gate. The exposure was to the *build*, not the runtime: provisioning a host
required cloning Jito-owned repositories. This fork removes that exposure — the
kill switch is gone, the protos are vendored in-tree, and every dependency
resolves from crates.io.

## Usage

```
raiku-shredstream-proxy forward-only \
  --src-bind-addr 0.0.0.0 \
  --src-bind-port 20000 \
  --dest-ip-ports 127.0.0.1:8001 \
  --multicast-device eth0
```

Every flag also reads from the equivalent `SCREAMING_SNAKE_CASE` environment
variable.

| Flag | Default | Purpose |
|---|---|---|
| `--src-bind-addr` | `0.0.0.0` | Address to listen on |
| `--src-bind-port` | `20000` | Port to listen on; `0` picks an ephemeral port |
| `--dest-ip-ports` | — | Comma-separated `ip:port` destinations |
| `--endpoint-discovery-url` | — | HTTP JSON endpoint returning destination IPs, set-unioned with `--dest-ip-ports` |
| `--discovered-endpoints-port` | — | Port to use for discovered hosts; must be paired with the URL |
| `--multicast-device` | `doublezero1` | Device used for multicast route discovery |
| `--multicast-bind-ip` | auto | Multicast group to join; otherwise parsed from `ip --json route show dev <device>` |
| `--multicast-subscribe-port` | `20001` | Port for multicast shreds |
| `--metrics-report-interval-ms` | `15000` | Stats logging interval |
| `--debug-trace-shred` | `false` | Log and time `TraceShred` packets |
| `--num-threads` | up to 4 | Listener/forwarder thread count |

At least one of `--dest-ip-ports` or `--endpoint-discovery-url` is required.
`--endpoint-discovery-url` and `--discovered-endpoints-port` must be set together.

Multicast route discovery shells out to `ip` (`iproute2`), which is why the
runtime Docker image installs it.

## How Raiku deploys it

Configured by the `shred_forwarder` Ansible role in the `infrastructure` repo.
Astralane unicasts raw shreds to `udp/20000` on a mainnet validator, firewalled to
their relay egress IPs; this binary relays them to `127.0.0.1:<live TVU port>`, so
shreds that reach Astralane ahead of turbine enter replay early and the duplicates
are dropped by the validator's shred dedup. A wrapper script resolves the TVU port
from `agave-validator contact-info` and restarts the unit when that port moves.

## Building

Requires the Rust toolchain pinned in `rust-toolchain.toml` (1.84).

```bash
cargo build --release --bin raiku-shredstream-proxy
cargo clippy --all-features --all-targets --tests -- -D warnings
cargo test --all-features --locked
```

No git submodules and no git dependencies — a plain `git clone` is enough.
