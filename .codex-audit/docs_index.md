# Docs Index

| Path | Purpose | Promises | Notes |
|---|---|---|---|
| `README.md` | Monorepo build, test, benchmark, and release entrypoint | Workspace is buildable/testable from source; debug builds are not for production validators | High-signal operator/developer orientation |
| `Cargo.toml` | Workspace inventory and package metadata | The repo is a broad validator-client monorepo, not a single binary | Useful for subsystem mapping in later phases |
| `SECURITY.md` | Security reporting, incident handling, scope boundaries | Critical issues have a private response path; some surfaces are explicitly out of scope or operator-trusted; built-in RPC is not for direct untrusted exposure | Important negative guarantees and trust assumptions |
| `RELEASE.md` | Release branch, channel, and patch workflow | `master`/edge is least stable, beta/stable are progressively more soaked | Reliability depends on branch/channel choice |
| `docs/README.md` | Scope and build flow for docs site | This repo's docs focus on Agave validator behavior; protocol docs are split to another repo | Important authority boundary |
| `docs/src/operations/setup-a-validator.md` | First-time validator setup | Validator setup assumes trusted local key handling, non-root operation, and secure withdrawer separation | Operator workflow and authority handling |
| `docs/src/operations/guides/validator-start.md` | Validator bootstrap and runtime flags | `--known-validator`/`--only-known-rpc` reduce malicious snapshot risk; ledger retention can be bounded with `--limit-ledger-size` | Bootstrap trust and storage tradeoffs |
| `docs/src/operations/validator-or-rpc-node.md` | Mode-selection guidance for operators | Consensus and RPC roles have different incentives, reward models, and operational expectations | Useful boundary before mapping service slices |
| `docs/src/operations/setup-an-rpc-node.md` | RPC-specific deployment guidance | Full RPC should usually not share a consensus node; RPC history may depend on Bigtable; RPC should not be exposed directly without protections | Clear service-mode split |
| `docs/src/operations/best-practices/security.md` | Operator security baseline | Do not store withdrawer keys on validator hosts; do not run validator as root | Human/operator safeguards are part of system reliability |
| `docs/src/operations/guides/validator-failover.md` | Planned failover procedure | Two-node failover can avoid downtime and safety issues if identity/tower choreography is followed | Recovery path worth mapping to code later |
| `docs/src/implemented-proposals/tower-bft.md` | Consensus lockout and rollback design | Votes should converge with increasing rollback cost and threshold checks | Core safety/finality design promise |
| `docs/src/implemented-proposals/reliable-vote-transmission.md` | Vote delivery and encoding design | Votes should be eventually delivered and encoded despite leader churn or flooding | Uses explicit "eventual" semantics |
| `docs/src/implemented-proposals/repair-service.md` | Missing-shred and orphan recovery design | Repair makes best-effort replay progress within the current verifiable epoch | Uses explicit "best effort" semantics |
| `docs/src/implemented-proposals/commitment.md` | RPC commitment metric design | Validators expose stake/confirmation observations for client consistency decisions | Client-facing observability promise |
| `docs/src/implemented-proposals/durable-tx-nonces.md` | Extended transaction lifetime / replay resistance | Durable nonce accounts let clients sign later without making the nonce reusable | Explicit anti-replay contract and runtime promise |
| `docs/src/implemented-proposals/snapshot-verification.md` | Intended snapshot verification design | Snapshot account state should be verifiable against voted hash state | Appears to conflict with current security-scope caveats and needs code validation |
