# Audit Slices

## Selection Summary

- Source map: `.codex-audit/system_map.md` and `.codex-audit/effect_graph.jsonl`
- Candidate count: 6
- Selected count: 5
- Recommended first slice: `bootstrap-snapshot-acceptance`
- Reason: This slice sits on the largest unresolved trust boundary in the current map. It combines durable state restoration, cross-node authority, ambiguous completion, operator configuration, and an explicit contradiction between aspirational snapshot-verification docs and current security caveats.

## Ranked Slices

| Rank | Slice | Protected promise | Main risk | Evidence | Recommended next command |
| ---- | ----- | ----------------- | --------- | -------- | ------------------------ |
| 1 | Bootstrap snapshot acceptance and restore | Bootstrap should prefer trusted validators and restore correct state | Authority confusion between known-validator gossip hashes, peer selection, archive acceptance, and local restore | `validator/src/bootstrap.rs:818-851`; `validator/src/bootstrap.rs:858-889`; `validator/src/bootstrap.rs:891-909`; `validator/src/bootstrap.rs:925-1000`; `validator/src/bootstrap.rs:1028-1056`; `runtime/src/serde_snapshot.rs:514-580`; `runtime/src/serde_snapshot.rs:842-872`; `SECURITY.md:171-173` | `worksheet bootstrap-snapshot-acceptance` |
| 2 | Vote durability, tower persistence, and failover handoff | Tower continuity should prevent unsafe restart/failover voting | Ambiguous completion between vote send, gossip publication, tower save, and actually encoded ledger state | `core/src/voting_service.rs:140-162`; `gossip/src/cluster_info.rs:827-903`; `core/src/consensus/tower_storage.rs:93-118`; `core/src/consensus/tower_storage.rs:178-219`; `validator/src/commands/set_identity/mod.rs:43-47`; `docs/src/operations/guides/validator-failover.md:72-103` | `worksheet vote-tower-failover` |
| 3 | Repair request to repaired-shred acceptance | Repair should provide best-effort replay catch-up within the verifiable epoch | Network ambiguity and remote-peer authority over missing/orphan/duplicate repair paths | `core/src/repair/repair_service.rs:360-460`; `core/src/repair/repair_service.rs:618-660`; `core/src/repair/repair_service.rs:1088-1136`; `ledger/src/blockstore.rs:1930-1998`; `docs/src/implemented-proposals/repair-service.md` | `worksheet repair-ingest-path` |
| 4 | RPC history retention, purge, and fallback semantics | Historical RPC completeness is conditional, not absolute | Clients may not know whether absence means prune, disablement, or true miss | `ledger/src/blockstore_cleanup_service.rs:47-140`; `ledger/src/blockstore.rs:3883-3960`; `ledger/src/blockstore/blockstore_purge.rs:390-430`; `validator/src/commands/run/args/json_rpc_config.rs:72-94`; `docs/src/operations/setup-an-rpc-node.md:57-61` | `worksheet rpc-history-retention` |
| 5 | Durable nonce lifecycle and replay protection | Durable nonces should extend tx lifetime without allowing nonce reuse | Runtime/client completion ambiguity around nonce validation and mutation ordering | `programs/system/src/system_processor.rs:410-476`; `runtime/src/bank/check_transactions.rs:189-236`; `docs/src/implemented-proposals/durable-tx-nonces.md` | `worksheet durable-nonce-lifecycle` |

## Slice 1: Bootstrap Snapshot Acceptance And Restore

### Boundary

In scope:

- discovery of snapshot hashes from known validators
- filtering peer snapshot hashes against known-validator-published hashes
- operator trust controls via `--known-validator`, `--only-known-rpc`, and expected-genesis checks
- transition from accepted snapshot metadata to local restore inputs
- full/incremental snapshot deserialization and bank reconstruction acceptance boundary

Out of scope:

- full consensus correctness after the node is already replaying normally
- generic ledger download performance
- unrelated RPC service exposure issues
- deep auditing of every accounts-db storage primitive

### Protected Promise

Bootstrap should be constrained to trusted peers when possible, and snapshot-based restart should restore state that the validator can safely treat as local truth.

### Why This Slice Matters

This is the clearest map-level contradiction in the repo today. The operator docs and bootstrap code present known-validator hash matching as a trust control, while `SECURITY.md` still treats malicious snapshots as an area with significant historical shortcomings. This slice determines whether startup is fail-open or fail-closed when trust signals are incomplete, conflicting, or stale.

### Durable State

- downloaded snapshot archives
- serialized snapshot streams and accounts-db fields
- reconstructed local `Bank` / `AccountsDb`
- persistent accounts/snapshot directories

### Side Effects

- querying peers and gossip tables for snapshot-hash metadata
- selecting or rejecting candidate peers
- accepting archive material as startup input
- reconstructing local bank state from snapshot contents

### Authority Boundaries

- known-validator-published gossip hashes vs arbitrary peer advertisements
- expected genesis hash/operator config vs default acceptance behavior
- snapshot deserialization/reconstruction logic vs network-level correctness claims

### Uncertainty Points

- gossip incompleteness while waiting for known-validator hashes
- same-slot conflicting hashes where first-seen values win
- uncertainty about what is checked after peer selection but before local restore
- ambiguity between "hash match" and "archive correctness"

### Likely Failure Semantics

- Omission: known-validator hashes never arrive or archive validation path skips necessary checks.
- Duplication: multiple peers advertise the same snapshot line and compete as acceptable sources.
- Reordering: first-seen same-slot hash influences the known-hash set before conflicting evidence appears.
- Delay: bootstrap may sleep/retry while gossip populates.
- Partial completion: some known validators are observed, enough to proceed, but the trust set remains incomplete.
- Stale view: CRDS snapshot-hash data may lag the actual peer state.
- Corruption: accepted archive contents may still be inconsistent or malicious despite matching metadata.
- Overload: bootstrap timing, port reachability, or peer scarcity may narrow acceptable choices.
- Human/operator error: weak `--known-validator` configuration or unsafe defaults.
- Ambiguous completion: operator may not know whether startup accepted a merely matching snapshot or a fully validated one.

### Evidence

- Docs: `docs/src/operations/guides/validator-start.md`; `docs/src/implemented-proposals/snapshot-verification.md`; `SECURITY.md`
- Files: `validator/src/bootstrap.rs`; `validator/src/commands/run/args.rs`; `runtime/src/serde_snapshot.rs`; `runtime/src/accounts_background_service/pending_snapshot_packages.rs`
- Modules/types/functions:
  - `validator::bootstrap::get_peer_snapshot_hashes`
  - `validator::bootstrap::get_snapshot_hashes_from_known_validators`
  - `validator::bootstrap::build_known_snapshot_hashes`
  - `runtime::serde_snapshot::fields_from_streams`
  - `runtime::serde_snapshot::reconstruct_bank_from_fields`
- Effect graph edges:
  - `edge-003`
  - `edge-004`
  - `edge-009`
  - `edge-010`

### Open Questions

- What exact code path verifies downloaded archive contents after peer selection?
- Is acceptance fail-open when known-validator hash evidence is sparse, conflicting, or stale?
- How do full/incremental snapshot interactions constrain what "matching snapshot hash" actually proves?

### Why This Should / Should Not Be First

This should be first because it resolves the highest-value trust ambiguity in the current map without requiring a whole-consensus audit first. It also sets the baseline for every later slice that assumes the node restarted from valid local state.

This should not be first only if bootstrap verification turns out to fan out into several unrelated sub-pipelines that must be separated immediately. That is currently a manageable risk.

### Recommended Next Command

`worksheet bootstrap-snapshot-acceptance`

## Slice 2: Vote Durability, Tower Persistence, And Failover Handoff

### Boundary

In scope:

- vote send path and CRDS publication
- local tower save/restore semantics
- restart/failover handoff using `set-identity` and `--require-tower`
- boundary between local vote continuity and cluster-encoded vote durability

Out of scope:

- full fork-choice proof or slashing analysis across the entire consensus implementation
- unrelated operator key-management practices beyond identity/tower handoff

### Protected Promise

Tower continuity and controlled failover should prevent the validator from resuming with an unsafe or stale local voting state.

### Why This Slice Matters

This slice protects the highest-value safety guarantee in the repo: not turning restart or failover into slashable or inconsistent vote behavior. It is also where local durable state, operator choreography, and ambiguous network completion intersect directly.

### Durable State

- signed tower file
- local identity path / key material
- on-chain vote-account state after votes are encoded

### Side Effects

- send vote transaction
- publish vote into gossip CRDS
- write tower file
- operator copies tower and flips identity during failover

### Authority Boundaries

- local tower file vs cluster-encoded vote state
- operator failover actions vs validator-side guards
- CRDS vote publication vs actual ledger inclusion

### Uncertainty Points

- a vote may be sent but not durably encoded yet
- tower file may exist but be stale for standby takeover
- operator may complete identity switch before durable tower continuity is assured

### Likely Failure Semantics

- Omission: vote send fails or tower copy never arrives.
- Duplication: both nodes may transiently act on the same voting identity.
- Reordering: identity switch precedes tower arrival or latest vote encoding.
- Delay: leader encoding lags local vote publication.
- Partial completion: tower saved locally but vote not encoded, or vice versa.
- Stale view: standby restores an old tower.
- Corruption: mangled or mismatched tower file.
- Overload: leader/network conditions delay durable vote encoding.
- Human/operator error: skipping `--require-tower` or mis-sequencing failover.
- Ambiguous completion: local node cannot immediately prove a sent vote became ledger truth.

### Evidence

- Docs: `docs/src/implemented-proposals/tower-bft.md`; `docs/src/operations/guides/validator-failover.md`
- Files: `core/src/voting_service.rs`; `gossip/src/cluster_info.rs`; `core/src/consensus.rs`; `core/src/consensus/tower_storage.rs`; `validator/src/commands/set_identity/mod.rs`
- Modules/types/functions:
  - `VotingService`
  - `ClusterInfo::push_vote`
  - `Tower::save`
  - `Tower::restore`
  - `FileTowerStorage::store`
  - `set_identity::execute`
- Effect graph edges:
  - `edge-001`
  - `edge-002`
  - `edge-014`

### Open Questions

- What is the strongest local proof that a sent vote is durably encoded?
- Beyond presence and signature, what makes a restored tower "fresh enough" for safe takeover?

### Why This Should / Should Not Be First

This should be near the front because it guards safety-critical restart and failover behavior. It is slightly less attractive than the bootstrap slice as a first worksheet because the cluster-side proof of durable vote encoding likely fans into broader replay and vote-account observation logic.

### Recommended Next Command

`worksheet vote-tower-failover`

## Slice 3: Repair Request To Repaired-Shred Acceptance

### Boundary

In scope:

- generation of repair work
- peer selection and request emission
- outstanding repair tracking
- insertion/acceptance of repaired shreds into blockstore
- duplicate/orphan repair paths where local state can change after remote input

Out of scope:

- full data-plane/turbine performance tuning
- generic network stack internals not tied to repair acceptance

### Protected Promise

Repair should provide bounded, best-effort replay catch-up without confusing remote assertions for local durable truth.

### Why This Slice Matters

This is the clearest bounded degraded-mode workflow in the map. It crosses a remote-peer trust boundary, uses best-effort semantics explicitly, and feeds directly into durable ledger state.

### Durable State

- blockstore shreds and slot metadata
- orphan / duplicate-slot state
- outstanding repair metadata

### Side Effects

- send repair requests
- receive repaired shreds
- insert repaired shreds into local blockstore

### Authority Boundaries

- peer claims about missing data vs local verification/insertion
- local outstanding-repair bookkeeping vs actual network completion

### Uncertainty Points

- sender does not know whether peer received or acted on repair request
- different peers may serve different useful data
- duplicate repair paths may rewrite local understanding of which slot version is acceptable

### Likely Failure Semantics

- Omission: no response or no useful shred set.
- Duplication: repeated requests/responses.
- Reordering: orphan/duplicate repairs arrive in non-causal order.
- Delay: replay waits while repairs are outstanding.
- Partial completion: some holes close, others remain.
- Stale view: peer `EpochSlots` signal is outdated.
- Corruption: wrong alternate slot material is accepted or needs correction.
- Overload: batch send failure or repair backlog.
- Human/operator error: overly strict repair validator restrictions.
- Ambiguous completion: request issuance is not proof of recovery.

### Evidence

- Docs: `docs/src/implemented-proposals/repair-service.md`
- Files: `core/src/repair/repair_service.rs`; `ledger/src/blockstore.rs`
- Modules/types/functions:
  - `RepairService`
  - `RepairInfo`
  - `OutstandingShredRepairs`
  - `Blockstore::do_insert_shreds`
- Effect graph edges:
  - `edge-005`
  - `edge-006`

### Open Questions

- Which persisted or observable markers distinguish "still outstanding" from "repair path exhausted"?
- Where is the narrowest acceptance boundary for repaired alternate-slot material?

### Why This Should / Should Not Be First

This should be early because it is rich in omission/duplication/delay semantics and is more bounded than consensus as a whole. It should not be first ahead of bootstrap because its trust model is less contradictory in the current docs and map.

### Recommended Next Command

`worksheet repair-ingest-path`
