# Repo Profile

## Purpose

Agave is the Anza-maintained Solana validator client monorepo. The workspace builds the validator, RPC node, CLI tooling, runtime, ledger/account storage, networking, test harnesses, and operator documentation for running or integrating with the client (`Cargo.toml`, `README.md`, `docs/README.md`).

The repository-specific docs are primarily about the Agave validator client, while protocol-wide Solana documentation is explicitly split out into another repository. That makes this repo authoritative for many client and operator behaviors, but not for every protocol guarantee (`docs/README.md`).

## Major subsystems

- Consensus and validator orchestration: `validator`, `core`, `vote`, `poh`, `gossip`, `turbine`, `streamer`, `repair`-related logic, plus operator flows for startup and failover (`Cargo.toml`, `docs/src/implemented-proposals/tower-bft.md`, `docs/src/implemented-proposals/reliable-vote-transmission.md`, `docs/src/implemented-proposals/repair-service.md`, `docs/src/operations/guides/validator-failover.md`).
- Runtime and program execution: `runtime`, `program-runtime`, `svm*`, `syscalls`, `precompiles`, built-in programs, and transaction/account execution surfaces (`Cargo.toml`).
- Durable state and bootstrap: `ledger`, `accounts-db`, `snapshots`, `storage-bigtable`, `storage-proto`, `genesis`, and validator bootstrap flows around known validators and snapshot sources (`Cargo.toml`, `docs/src/operations/guides/validator-start.md`, `docs/src/implemented-proposals/snapshot-verification.md`).
- Client and API surfaces: `rpc`, `rpc-client*`, `pubsub-client`, `transaction-status*`, `account-decoder*`, `cli*`, `install`, `keygen`, and docs for RPC-node operation (`Cargo.toml`, `docs/src/operations/setup-an-rpc-node.md`).
- Ops, release, and observability: release channels, backports, CI/release automation, benchmarking, docs site generation, and monitoring/watchtower utilities (`Cargo.toml`, `README.md`, `RELEASE.md`, `docs/README.md`).

## Build/test commands

- Build the workspace with `./cargo build`; the top-level README warns the default debug build is not suitable for production validators (`README.md`).
- Run the main test suite with `./cargo nextest run --profile ci --cargo-profile ci --config-file .config/nextest.toml` (`README.md`).
- Run benchmarks with `cargo +nightly bench` (`README.md`).
- Generate coverage with `scripts/coverage.sh` (`README.md`).
- Build validator docs from `docs/` with `./build.sh`; this requires Docker and also generates CLI usage docs and SVG assets (`docs/README.md`).

## Risk-bearing areas

- Consensus safety and liveness: Tower lockouts, fork choice, threshold checks, vote propagation, and validator failover all protect against safety violations and stalled voting, but they are sensitive to restart handling and state continuity (`docs/src/implemented-proposals/tower-bft.md`, `docs/src/implemented-proposals/reliable-vote-transmission.md`, `docs/src/operations/guides/validator-failover.md`).
- Bootstrap trust and state correctness: operator docs recommend `--known-validator` and `--only-known-rpc` to reduce malicious snapshot risk, while security docs still describe snapshot trust as incomplete (`docs/src/operations/guides/validator-start.md`, `SECURITY.md`, `docs/src/implemented-proposals/snapshot-verification.md`).
- RPC exposure and history completeness: the built-in RPC is not intended for direct untrusted exposure, and truncated local ledgers push historical dependence onto Bigtable or other external storage (`docs/src/operations/setup-an-rpc-node.md`, `SECURITY.md`).
- Key management and authority separation: withdrawer keys and validator identities are operator-controlled authorities whose misuse can directly compromise funds or validator control (`docs/src/operations/setup-a-validator.md`, `docs/src/operations/best-practices/security.md`).
- Release and backport flow: stable/beta/edge channels deliberately trade freshness for soak time, so branch selection is part of the reliability boundary (`RELEASE.md`).
- Trusted extensions: security policy explicitly treats Geyser plugins and `scheduler-bindings` external processes as operator-trusted surfaces outside some project guarantees (`SECURITY.md`).

## Unknowns

- Which design docs under `docs/src/implemented-proposals/` remain fully normative versus historical background needs validation from code in later audit phases.
- The exact code-level enforcement points for snapshot trust, tower persistence, failover safety, and RPC exposure are not mapped yet.
- Protocol-wide guarantees are partially documented outside this repository, so some promises in this workspace are client-local while others are inherited from upstream Solana protocol docs (`docs/README.md`).
