# Open Questions

1. Which documents under `docs/src/implemented-proposals/` are still normative for the current Agave codebase, and which are historical design notes that now diverge from implementation?

2. Snapshot trust is currently ambiguous. `docs/src/implemented-proposals/snapshot-verification.md` describes a verifiable snapshot model, while `SECURITY.md` says malicious snapshots are historically trust-on-first-use and still have major consistency/verifiability shortcomings. What does the current code actually guarantee during snapshot bootstrap?

3. Where are the code-level enforcement points for bootstrap trust decisions such as `--known-validator`, `--only-known-rpc`, expected genesis hash checks, and snapshot acceptance? Likely areas include `validator`, `ledger`, `snapshots`, and related startup code, but this is not mapped yet.

4. Which persisted artifacts are safety-critical across restart and failover? The docs imply that tower files and identity switching are essential, but the exact invariant and repair path still need code-backed mapping (`docs/src/operations/guides/validator-failover.md`).

5. What are the precise completeness guarantees of RPC history under combinations of `--limit-ledger-size`, `--enable-rpc-transaction-history`, secondary indexes, and Bigtable-backed history?

6. What reliability properties are enforced when operator-trusted integrations misbehave, especially Geyser plugins and `scheduler-bindings` external processes that security policy treats as outside some project guarantees (`SECURITY.md`)?

7. Which reliability promises are client-specific to Agave versus protocol-level promises maintained in external Solana documentation (`docs/README.md`)? This boundary matters before admitting any future finding.

8. What source of truth should this audit treat as authoritative for release-channel risk: the branch/channel semantics in `RELEASE.md`, the operator docs, or current CI/release automation in workflows and scripts?
