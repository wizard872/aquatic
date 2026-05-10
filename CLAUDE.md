# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`aquatic` is a high-performance open BitTorrent tracker, implemented as a Cargo workspace of three protocol-specific server binaries plus shared libraries. All tracker state is held in memory (no database).

| Server | Protocol | OS |
|---|---|---|
| `aquatic_udp` | BitTorrent over UDP (BEP 015) | Unix-like |
| `aquatic_http` | BitTorrent over HTTP(S) | Linux 5.8+ (uses `glommio` / io_uring) |
| `aquatic_ws` | WebTorrent over WS(S) | Linux 5.8+ (uses `glommio` / io_uring) |

The `combined_binary` crate dispatches to one of the three based on its first argument (`udp` / `http` / `ws`).

## Build & run

Use the recommended SIMD env *before* building/running for ~best performance — it sets `RUSTFLAGS=-C target-cpu=native` minus AVX-512 (which throttles many CPUs):

```sh
. ./scripts/env-native-cpu-without-avx-512
```

Build a single server: `cargo build --release -p aquatic_udp` (or `aquatic_http`, `aquatic_ws`).

Each binary uses the same CLI shape:
- `-p` prints a default TOML config to stdout (redirect to a file, edit, then pass with `-c`)
- `-c <path>` runs with the given config

`scripts/run-aquatic-{udp,http,ws}.sh` are dev shortcuts that compile with the `release-debug` profile and forward arguments. `scripts/run-load-test-{udp,http,ws}.sh` do the same for the load-test crates.

UDP has an experimental io_uring backend behind a feature flag: `cargo build --release -p aquatic_udp --features io-uring` (Linux 6.0+).

`scripts/gen-tls.sh` generates a self-signed cert/key under `./tmp/tls/` for local TLS testing of HTTP/WS.

## Test

CI uses a dedicated profile that builds with `release` codegen but no LTO, so tests run at production-like speed without long link times:

```sh
# Full workspace tests (matches CI)
cargo test --profile "test-fast" --workspace

# Single crate
cargo test --profile "test-fast" -p aquatic_udp

# UDP with io_uring backend (separate CI job)
cargo test --profile "test-fast" -p aquatic_udp --features io-uring

# Single test by name
cargo test --profile "test-fast" -p aquatic_udp -- requests_responses
```

End-to-end file-transfer tests run inside a container (`scripts/ci-file-transfers.sh` builds `docker/ci.Dockerfile` and runs it) — they need `--privileged` / raised memlock and won't work outside Docker.

The `bench` and regular `test` profiles inherit from `release-debug` (release + debuginfo); use `test-fast` for iteration speed.

## Architecture

### Shared crates
- `common` — cross-server utilities: access lists, CLI/config bootstrap (`run_app_with_cli_and_config`), `PrivilegeDropper`, optional rustls config, optional Prometheus endpoint, `WorkerType` enum, `ValidUntil` / `ServerStartInstant` (32-bit time-since-start used everywhere instead of `Instant` to shrink per-peer memory).
- `udp_protocol`, `http_protocol`, `ws_protocol` — wire-format parse/serialize. UDP uses `zerocopy` for hot-path zero-allocation parsing.
- `peer_id` — decode BitTorrent client info from peer IDs.
- `toml_config` + `toml_config_derive` — derive macro for TOML configs that round-trip with comments (used by every server's `-p` output).

### Server thread model

Every server `run()` follows the same shape: spawn worker threads, then poll their `JoinHandle`s — if **any** worker exits or panics, the whole process exits with an error (see `aquatic_udp::run` in `crates/udp/src/lib.rs`).

**`aquatic_udp`** — uses `mio` (or optionally `io_uring`) directly, no async runtime:
- N **socket workers** (one per `socket_workers` config; defaults to `available_parallelism()`). Each holds its own UDP socket(s) — one for IPv4 and/or one for IPv6 depending on `network.use_ipv4` / `network.use_ipv6`. **All workers share one `TorrentMaps`** via `Arc<[RwLock<TorrentMapShard>]>` (16 shards, sharded by `info_hash[0] % 16` — `swarm.rs`). Workers contend on per-shard `RwLock`s, not on a single global lock. Per-torrent peer state is behind another `RwLock<PeerMap>` inside `TorrentData`.
- 1 **cleaning** thread (periodic torrent/peer expiry + access-list refresh).
- 1 **statistics** thread (if `statistics.active()`).
- 1 **prometheus** thread (if enabled).
- 1 **signals** thread — `SIGUSR1` triggers access-list reload (and TLS cert reload in HTTP/WS).
- A `ConnectionValidator` (BLAKE3-keyed) replaces the old per-IP connection map; spoofing is mitigated cryptographically rather than by tracking state.

**`aquatic_http` / `aquatic_ws`** — built on `glommio` (thread-per-core, io_uring under the hood):
- N **socket workers** + M **swarm workers**, connected by a `glommio` `MeshBuilder` channel. Socket workers parse requests and forward them to the swarm worker that owns the relevant info hash; swarm workers hold the torrent maps and produce responses.
- TLS, when enabled, uses `rustls` via `ArcSwap` so `SIGUSR1` can hot-swap the certificate.
- Reverse-proxy mode is supported (`network.runs_behind_reverse_proxy`).

### Conventions worth knowing
- Peers are indexed by `(packet source IP, announced port)` for UDP/HTTP, **not** by `peer_id` — this is intentional anti-spoofing and predates the current code. WS, by protocol nature, indexes by `peer_id`.
- Per-peer/per-connection lifetimes use `ValidUntil` (a `u32` of seconds since server start), not `Instant`. New time-bounded data should follow this pattern to keep memory small.
- **Announce response semantics** — peer **list** excludes the announcing peer (so the response doesn't have to be filtered). Seeder/leecher **counts** include the announcing peer based on its post-announce status, so they reflect the full swarm count for that info_hash. This is enforced in `PeerMap::announce` (UDP `swarm.rs`) and `TorrentData::upsert_peer_and_get_response_peers` (HTTP `workers/swarm/storage.rs`) via a small `adjust_counts_for_announcing_peer` helper applied right after `num_seeders_leechers()`. WS achieves the same by counting *after* `insert_or_update_peer` runs. Note: this contradicts the 0.9.0 CHANGELOG entry "don't include announcing peer" — the local change reverted that for count parity with `scrape`.
- CPU pinning support was removed in 0.9.0 — don't reintroduce it without discussion.
- `mimalloc` is the default global allocator on the binary crates (feature-gated, on by default).
- `deny.toml` is configured for `cargo-deny`; license/source policies are enforced there.

## Repo map

- `crates/{udp,http,ws}/src/workers/` — per-server socket / swarm / statistics worker code.
- `crates/udp/src/workers/socket/{mio,uring}/` — the two UDP socket backends behind the `io-uring` feature.
- `crates/udp/tests/` — integration tests including `requests_responses.rs`, `access_list.rs`, `invalid_connection_id.rs`.
- `crates/bencher/` — automated comparative benchmarking against other trackers.
- `documents/` — load-test PDFs/PNGs and the architecture SVG (`aquatic-architecture-2024.svg`).
- `TODO.md` and `CHANGELOG.md` are kept current; check `CHANGELOG.md` "Unreleased" before assuming current behavior matches an old README.
