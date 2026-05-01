# Reliability System Map

## Scope

This map covers the first-order reliability system behind the promises already extracted in `.codex-audit/promises.md`:

- validator voting and tower continuity
- ledger and RPC-history durability
- repair and replay catch-up
- bootstrap from known validators and snapshots
- durable transaction nonces
- RPC service mode boundaries
- operator-trusted plugin/export paths

This is a promise map, not a module inventory or deployment diagram.

## Map Summary

The main durable truth surfaces in Agave are:

- local ledger/blockstore RocksDB under the validator ledger path
- local accounts storage and snapshot artifacts used for restart/rebuild
- the signed tower file used to re-establish a validator's local vote lockout state
- on-chain account state, including nonce accounts and vote-account authority
- optional local RPC history/index columns that are explicitly prunable

The highest-risk side effects are:

- sending vote transactions and gossip votes without an immediate proof they were durably encoded
- accepting bootstrap material from peers based on snapshot hashes observed in gossip
- issuing repair requests and integrating repaired shreds asynchronously
- purging old ledger history under `--limit-ledger-size`
- streaming state changes to loaded plugins and whatever external stores they control

The main authority boundaries are:

- cluster consensus versus local node observation
- runtime/system-program validation versus client-submitted intent
- known-validator gossip hashes versus arbitrary bootstrap peers
- operator-controlled keys, tower files, and identity switching versus validator automation
- reverse proxy / surrounding deployment controls versus the built-in RPC service

## Promise-to-System Map

| Promise | Durable state | Side effect | Authority | Uncertainty | Evidence |
| ------- | ------------- | ----------- | --------- | ----------- | -------- |
| Tower BFT should make inconsistent rollback increasingly expensive and restart-safe for the validator. | Signed tower file `tower-1_9-<pubkey>.bin`; on-chain vote state after votes land in blocks | Vote tx sent to TPU/leader path and vote also published to CRDS gossip | Local tower file is authoritative for restart-local lockout continuity; cluster consensus is authoritative for whether the vote actually landed | A sent vote may not be durably encoded yet; restart with missing/mangled tower is explicitly dangerous | `core/src/consensus/tower_storage.rs:93-118`; `core/src/consensus/tower_storage.rs:178-219`; `core/src/consensus.rs:1682-1689`; `gossip/src/cluster_info.rs:884-903`; `core/src/voting_service.rs:140-162`; `docs/src/implemented-proposals/tower-bft.md`; `docs/src/operations/guides/validator-failover.md:72-103` |
| Vote propagation is designed for eventual delivery and eventual encoding. | Durable once encoded into ledger/blockstore; before that, only transient CRDS and network state | Send vote over TPU path, publish vote into CRDS, future leaders observe and may encode | Leader/cluster replay is authoritative for encoding; local CRDS publication is only propagation state | Network send has no durable ack; CRDS presence does not prove final encoding | `core/src/voting_service.rs:140-162`; `gossip/src/cluster_info.rs:827-903`; `docs/src/implemented-proposals/reliable-vote-transmission.md` |
| Repair is best-effort within the current verifiable epoch. | Blockstore slot metadata, orphan tracking, duplicate-slot tracking, outstanding repair metadata | Emit repair UDP requests, receive repaired shreds, insert repaired shreds into blockstore | Remote peers are authoritative for what they serve; local shred verification/blockstore insertion is authoritative for what the node accepts | Requests can be lost, delayed, duplicated, or satisfied by stale peers; outcome is intentionally best-effort | `ledger/src/blockstore.rs:238-283`; `ledger/src/blockstore.rs:1930-1998`; `core/src/repair/repair_service.rs:360-460`; `core/src/repair/repair_service.rs:618-660`; `core/src/repair/repair_service.rs:1088-1136`; `docs/src/implemented-proposals/repair-service.md` |
| Bootstrap should prefer trusted validators and only accept matching snapshot hashes when configured. | Downloaded local snapshot archives and restored local state; CRDS snapshot-hash observations | Query peers, compare peer snapshot hashes to known-validator hashes, then accept/download bootstrap material | Known-validator-published snapshot hashes and expected genesis hash gate acceptance; arbitrary peers are not authoritative alone when trust is configured | Gossip may be incomplete; validators may disagree on same-slot hashes; current security policy still treats malicious snapshots as a weak area | `validator/src/commands/run/args.rs:608-618`; `validator/src/bootstrap.rs:get_peer_snapshot_hashes`; `validator/src/bootstrap.rs:858-889`; `validator/src/bootstrap.rs:891-909`; `validator/src/bootstrap.rs:925-1000`; `validator/src/bootstrap.rs:1028-1056`; `docs/src/operations/guides/validator-start.md:258-262`; `SECURITY.md:171-173` |
| Snapshot-based restart should reconstruct accounts/bank state, but exact trust guarantees remain ambiguous. | Full/incremental snapshot archives; serialized `AccountsDbFields`; rebuilt `Bank` and `AccountsDb` | Package snapshots, queue newer snapshot packages, deserialize snapshot streams, reconstruct bank/accounts on restart | Snapshot serialization/deserialization code is authoritative for what is loaded locally; docs do not fully settle whether that proves network-correctness | Restart can depend on incomplete or conflicting snapshot evidence; incremental/full interactions are subtle | `runtime/src/accounts_background_service/pending_snapshot_packages.rs:19-70`; `runtime/src/serde_snapshot.rs:95-110`; `runtime/src/serde_snapshot.rs:514-580`; `runtime/src/serde_snapshot.rs:842-872`; `accounts-db/src/accounts_db.rs:961-988`; `docs/src/implemented-proposals/snapshot-verification.md`; `SECURITY.md:171-173` |
| Durable transaction nonces extend transaction lifetime without allowing nonce reuse. | Nonce account state in accounts storage/snapshots; recent blockhash sysvar account | Initialize, advance, withdraw, or authorize nonce account; validate transactions against durable nonce state | Runtime/system program is authoritative for nonce-account mutation and transaction acceptance | Client can submit a tx without immediate certainty of final outcome; runtime has special handling because rollback semantics around nonce advancement are subtle | `programs/system/src/system_processor.rs:410-476`; `runtime/src/bank/check_transactions.rs:189-236`; `docs/src/implemented-proposals/durable-tx-nonces.md` |
| RPC history is conditional, not absolute, and can be delegated to Bigtable. | Local `transaction_status_cf` and `address_signatures_cf` plus optional account indexes; external Bigtable if enabled | Store tx-status metadata, serve RPC queries, purge old local history, optionally fall back to Bigtable | Local blockstore columns are authoritative for locally retained history; Bigtable is authoritative only for fallback results it returns | Local pruning can remove needed history; some RPC completeness depends on flags and external storage | `ledger/src/blockstore.rs:265-272`; `ledger/src/blockstore.rs:3883-3960`; `ledger/src/blockstore/blockstore_purge.rs:390-430`; `validator/src/commands/run/args/json_rpc_config.rs:72-94`; `validator/src/commands/run/args/json_rpc_config.rs:163-167`; `validator/src/commands/run/args.rs:881-908`; `docs/src/operations/setup-an-rpc-node.md:14-26`; `docs/src/operations/setup-an-rpc-node.md:57-61` |
| `--limit-ledger-size` intentionally trades local durability for bounded disk usage. | Blockstore shreds and metadata below the configured retention limit | Cleanup thread purges old ledger data from local storage | Local cleanup service and validator config are authoritative for what is retained locally | Operators may not know a queried history item has already been purged unless external fallback is configured | `validator/src/commands/run/execute.rs:444-458`; `ledger/src/blockstore_cleanup_service.rs:47-140`; `docs/src/operations/guides/validator-start.md:312-321`; `docs/src/operations/setup-an-rpc-node.md:59-61` |
| Validator failover is safe only if operator-controlled identity and tower continuity are preserved. | Tower file in ledger path; identity key material; symlink/operator shell state | `set-identity`, copy tower file, relink identity, resume voting on standby | Operator action plus `--require-tower` guard is authoritative for failover intent; validator enforces only part of the choreography | Copy/switch may complete partially; tower may be missing on standby; both nodes may transiently disagree about active identity | `validator/src/commands/set_identity/mod.rs:43-47`; `validator/src/commands/set_identity/mod.rs:53-87`; `docs/src/operations/guides/validator-failover.md:72-103` |
| Geyser and similar extensions are operator-trusted export surfaces, not core validator guarantees. | Plugin config files, loaded dynamic libraries, and whatever external store the plugin writes to | Runtime/account/slot/tx notifications forwarded to plugin callbacks | Plugin implementation and external sink are authoritative for downstream side effects, not the validator | Plugin calls may fail after core state has already changed; downstream export success is not the same as validator correctness | `geyser-plugin-interface/src/geyser_plugin_interface.rs:436-590`; `geyser-plugin-manager/src/geyser_plugin_service.rs:57-121`; `geyser-plugin-manager/src/accounts_update_notifier.rs:125-160`; `SECURITY.md:163-165` |

## Durable State

### Ledger Blockstore

- Purpose: Durable local record of shreds, slot metadata, roots, optimistic slots, dead slots, and optional RPC transaction history/indexes.
- Location: Ledger path RocksDB opened under `ledger_path.join(BLOCKSTORE_DIRECTORY_ROCKS_LEVEL)`.
- Writer: Validator replay, block ingestion, repair insertion, RPC metadata writers, and cleanup/purge logic.
- Reader: Replay, repair, RPC history queries, ledger tools, and bootstrap/restart paths.
- Authority: The local node is authoritative for what it has durably stored; cluster consensus is still authoritative for global truth.
- Crash behavior: Data that reached RocksDB persists across process restarts; in-memory working sets are not durable until committed.
- Evidence: `ledger/src/blockstore.rs:238-283`; `ledger/src/blockstore.rs:484-530`; `ledger/src/blockstore.rs:1930-1998`.

### Accounts Database And Snapshot Artifacts

- Purpose: Durable account state for runtime execution and restart/rebuild from full/incremental snapshots.
- Location: `AccountsDb` storages and snapshot archives/streams.
- Writer: Runtime execution, background snapshot packaging, snapshot serialization.
- Reader: Runtime transaction checks, startup restore, snapshot deserialization.
- Authority: Local runtime reconstruction logic is authoritative for what state is restored on this node.
- Crash behavior: Persisted storages and archives survive restart; pending in-memory snapshot packages do not unless written out.
- Evidence: `accounts-db/src/accounts_db.rs:871-988`; `runtime/src/accounts_background_service/pending_snapshot_packages.rs:19-70`; `runtime/src/serde_snapshot.rs:514-580`; `runtime/src/serde_snapshot.rs:842-872`.

### Tower File

- Purpose: Preserve local vote lockouts/root continuity across restart or failover.
- Location: `tower-1_9-<node_pubkey>.bin` in the configured tower path.
- Writer: `Tower::save()` via `FileTowerStorage::store()`.
- Reader: `Tower::restore()` via `FileTowerStorage::load()`.
- Authority: The tower file is locally authoritative for restart-time lockout continuity only when it matches the node pubkey and signature.
- Crash behavior: Writes go to `*.bin.new` then `rename()` to the stable filename, reducing torn-write exposure but without explicit `sync_all()`.
- Evidence: `core/src/consensus/tower_storage.rs:93-118`; `core/src/consensus/tower_storage.rs:153-219`; `core/src/consensus.rs:1682-1689`.

### Nonce Account State

- Purpose: Durable anti-replay state for transactions that outlive the normal recent-blockhash window.
- Location: Nonce account data in accounts storage plus recent blockhash sysvar state.
- Writer: System program nonce instructions and runtime blockhash updates.
- Reader: Runtime transaction validation and system-program instruction processing.
- Authority: Runtime/system program.
- Crash behavior: Once the account mutation is committed in bank/account state and later snapshotted/replayed, the nonce survives restart.
- Evidence: `programs/system/src/system_processor.rs:410-476`; `runtime/src/bank/check_transactions.rs:208-236`; `runtime/src/bank.rs:2581-2594`.

### RPC History And Account Index Columns

- Purpose: Support transaction-history and indexed account RPC queries from local storage.
- Location: `transaction_status_cf`, `address_signatures_cf`, related metadata columns, and optional account indexes.
- Writer: Blockstore transaction-status writers and validator configuration enabling history/indexes.
- Reader: RPC query paths and purge logic.
- Authority: Local node only, bounded by configured retention and indexing.
- Crash behavior: Persisted if written to blockstore; may later be explicitly purged.
- Evidence: `ledger/src/blockstore.rs:265-272`; `ledger/src/blockstore.rs:3883-3960`; `ledger/src/blockstore/blockstore_purge.rs:390-430`; `validator/src/commands/run/args/json_rpc_config.rs:72-94`; `validator/src/commands/run/args.rs:881-908`.

## Side Effects

### Vote Submission And Gossip Publication

- Trigger: Voting service creates a vote operation for a bank/tower update.
- External change: Vote transaction is sent toward TPU/leader path, and vote is inserted into local CRDS for gossip propagation.
- Commit/acceptance boundary: Durable acceptance happens only if a leader encodes the vote and replay eventually accepts it into ledger state.
- Retry behavior: Gossip refresh/push can republish; later leaders may encode votes already observed in CRDS.
- Ambiguous outcome: A sent or gossiped vote may have been observed, ignored, or not yet durably encoded.
- Evidence: `core/src/voting_service.rs:140-162`; `gossip/src/cluster_info.rs:827-903`; `docs/src/implemented-proposals/reliable-vote-transmission.md`.

### Repair Request And Repaired Shred Ingestion

- Trigger: Repair service identifies missing/orphan/duplicate-repair work.
- External change: UDP repair requests sent to peers; repaired shreds later inserted into local blockstore.
- Commit/acceptance boundary: The local node accepts the effect only after shred verification and blockstore insertion.
- Retry behavior: Outstanding repair requests are tracked and additional requests may be issued.
- Ambiguous outcome: Sender may not know if peer received, responded, or responded with useful data.
- Evidence: `core/src/repair/repair_service.rs:618-660`; `core/src/repair/repair_service.rs:1088-1136`; `ledger/src/blockstore.rs:1930-1998`; `docs/src/implemented-proposals/repair-service.md`.

### Snapshot Bootstrap Acceptance

- Trigger: Validator startup/bootstrap with peer RPC and snapshot discovery.
- External change: Peer snapshot hashes are gathered from CRDS, filtered through known-validator hashes, and a snapshot source may be selected for download/use.
- Commit/acceptance boundary: Local bootstrap accepts a peer snapshot only after matching it against the accepted known snapshot-hash set when configured.
- Retry behavior: Bootstrap waits and retries while gossip is incomplete.
- Ambiguous outcome: Missing gossip, conflicting same-slot hashes, or weak snapshot trust can leave the operator unsure whether the accepted snapshot is network-correct.
- Evidence: `validator/src/bootstrap.rs:818-851`; `validator/src/bootstrap.rs:858-889`; `validator/src/bootstrap.rs:891-909`; `validator/src/bootstrap.rs:925-1000`; `validator/src/bootstrap.rs:1028-1056`; `SECURITY.md:171-173`.

### Snapshot Packaging And Restore

- Trigger: Root advancement and background snapshot packaging, or startup restore from snapshot streams.
- External change: New full/incremental snapshot packages and archives are produced; restart reconstructs local bank/accounts state from those streams.
- Commit/acceptance boundary: Package queue ordering and deserialization/reconstruction logic determine what local state is accepted.
- Retry behavior: Newer snapshot packages overwrite older pending in-kind packages.
- Ambiguous outcome: Full/incremental interactions and trust in snapshot provenance remain partially ambiguous at the repo-doc level.
- Evidence: `runtime/src/accounts_background_service/pending_snapshot_packages.rs:19-70`; `runtime/src/serde_snapshot.rs:514-580`; `runtime/src/serde_snapshot.rs:842-872`; `accounts-db/src/accounts_db.rs:2245-2263`.

### Ledger Cleanup Purge

- Trigger: Cleanup interval elapses and root advancement plus `max_ledger_shreds` indicate purge is due.
- External change: Old local ledger/history data is removed from blockstore.
- Commit/acceptance boundary: Once purge runs and deletes rows, local historical availability is reduced until/unless external fallback exists.
- Retry behavior: Cleanup loop rechecks periodically; manual purge requests can also be registered.
- Ambiguous outcome: RPC callers may not distinguish "never had it" from "had it but purged it" without external context.
- Evidence: `ledger/src/blockstore_cleanup_service.rs:47-140`; `validator/src/commands/run/execute.rs:444-458`; `ledger/src/blockstore/blockstore_purge.rs:390-430`.

### Plugin Notification To External Consumers

- Trigger: Account/slot/transaction/entry changes during startup restore or normal runtime processing.
- External change: Plugin callbacks may write to external data stores or perform arbitrary downstream actions.
- Commit/acceptance boundary: The validator considers the callback invoked; downstream external durability is owned by the plugin, not the validator.
- Retry behavior: No repo-level general retry guarantee is evident from the notifier interface; callback errors are logged.
- Ambiguous outcome: Core validator state may commit even when plugin export fails.
- Evidence: `geyser-plugin-interface/src/geyser_plugin_interface.rs:471-590`; `geyser-plugin-manager/src/geyser_plugin_service.rs:57-121`; `geyser-plugin-manager/src/accounts_update_notifier.rs:125-160`; `SECURITY.md:163-165`.

## Authority Boundaries

### Cluster Consensus Versus Local Observation

- Initiator: Local validator votes, replays, and serves RPC.
- Authority: The cluster's accepted fork and encoded ledger state.
- What authority proves: Whether a vote, block, or transaction became part of the cluster-observed ledger.
- What authority does not prove: That a local send/gossip action completed just because it was attempted.
- Risk if confused: A node may treat propagation or local storage as if it implied final encoding or cluster agreement.
- Evidence: `docs/src/implemented-proposals/tower-bft.md`; `docs/src/implemented-proposals/reliable-vote-transmission.md`; `core/src/voting_service.rs:140-162`.

### Runtime/System Program Versus Client Intent

- Initiator: Client submits a transaction using a recent blockhash or durable nonce.
- Authority: Runtime transaction checks and system-program nonce instruction handlers.
- What authority proves: Whether the nonce account was valid, initialized, advanceable, and acceptable for the transaction.
- What authority does not prove: That the client can infer final result from mere submission.
- Risk if confused: Replay/fee/nonce semantics can be misunderstood as a client-side retry-safe API when they are really runtime-controlled.
- Evidence: `programs/system/src/system_processor.rs:410-476`; `runtime/src/bank/check_transactions.rs:189-236`; `docs/src/implemented-proposals/durable-tx-nonces.md`.

### Known Validators Versus Arbitrary Bootstrap Peers

- Initiator: Bootstrap logic selecting peer snapshots.
- Authority: Snapshot hashes published in gossip by configured known validators, plus expected genesis hash when provided.
- What authority proves: That a candidate peer snapshot matches an accepted hash set built from known validators.
- What authority does not prove: That the overall snapshot trust story is solved or that gossip itself was complete/correct.
- Risk if confused: Operators may overestimate snapshot verifiability and accept state on weaker evidence than intended.
- Evidence: `validator/src/commands/run/args.rs:608-618`; `validator/src/bootstrap.rs:818-851`; `validator/src/bootstrap.rs:858-889`; `SECURITY.md:171-173`.

### Operator-Controlled Failover State

- Initiator: Human/operator switching active identity between validator instances.
- Authority: Operator-managed keys, tower file copy, symlink state, and `--require-tower` guard.
- What authority proves: That the standby will only assume the staked identity when a saved tower exists if `--require-tower` is used.
- What authority does not prove: That the full operational choreography happened atomically or correctly across both hosts.
- Risk if confused: Split-brain voting, missing tower continuity, or partial failover steps.
- Evidence: `validator/src/commands/set_identity/mod.rs:43-47`; `validator/src/commands/set_identity/mod.rs:53-87`; `docs/src/operations/guides/validator-failover.md:72-103`.

### Built-In RPC Versus Deployment Perimeter

- Initiator: RPC clients and operator exposure decisions.
- Authority: Local replayed state for answers; reverse proxy / perimeter controls for abuse resistance.
- What authority proves: The validator can answer from local or configured fallback history; the built-in RPC itself does not prove internet-facing safety.
- What authority does not prove: Authentication, rate-limiting, or safe public exposure.
- Risk if confused: Treating the validator process as a complete public API perimeter.
- Evidence: `docs/src/operations/setup-an-rpc-node.md:14-26`; `SECURITY.md:174-180`; `validator/src/commands/run/args/json_rpc_config.rs:163-167`.

### Plugin Callback Boundary

- Initiator: Validator/plugin manager forwarding runtime events.
- Authority: Loaded plugin implementation and its configured downstream sink.
- What authority proves: Only that the callback was invoked or errored locally.
- What authority does not prove: That any external export/store is complete, ordered, or durable.
- Risk if confused: Assuming plugin-fed downstream systems are equivalent to validator truth.
- Evidence: `geyser-plugin-interface/src/geyser_plugin_interface.rs:471-590`; `geyser-plugin-manager/src/geyser_plugin_service.rs:57-121`; `geyser-plugin-manager/src/accounts_update_notifier.rs:125-160`; `SECURITY.md:163-165`.

## Uncertainty Points

### Vote Submission Without Durable Ack

- Scenario: A validator sends a vote transaction and updates CRDS, but the operator cannot directly observe durable ledger encoding yet.
- Why outcome is ambiguous: Send/gossip completion is separate from leader encoding and replay acceptance.
- Possible failure semantics:
  - omission: vote never reaches a leader
  - duplication: vote may be refreshed/republished
  - reordering: later leaders may encode before earlier opportunities
  - delay: eventual propagation is asynchronous
  - partial completion: CRDS update may happen without ledger inclusion
  - stale view: local node may not know cluster acceptance yet
  - corruption: mangled local tower on restart can make safe continuation unclear
  - overload: leader flooding is an explicit design concern
  - operator error: failover/restart with wrong tower
- Evidence: `core/src/voting_service.rs:140-162`; `gossip/src/cluster_info.rs:845-900`; `docs/src/implemented-proposals/reliable-vote-transmission.md`.
- Open question: What concrete code path or metric is treated as the strongest local proof that a previously sent vote became durably encoded?

### Best-Effort Repair Outcome

- Scenario: Repair service requests shreds from peers and tracks outstanding repairs.
- Why outcome is ambiguous: UDP send does not prove peer receipt or useful response, and peers may differ in what they can serve.
- Possible failure semantics:
  - omission: no response
  - duplication: repeated requests or repeated repaired shreds
  - reordering: orphan/duplicate corrections can arrive out of order
  - delay: request remains outstanding while replay stalls
  - partial completion: some holes repaired, others remain
  - stale view: peer advertises stale `EpochSlots`
  - corruption: wrong or alternate slot material may trigger duplicate-repair flows
  - overload: request batching can still fail partially
  - operator error: restricting repair validators incorrectly
- Evidence: `core/src/repair/repair_service.rs:618-660`; `core/src/repair/repair_service.rs:1088-1136`; `docs/src/implemented-proposals/repair-service.md`.
- Open question: Which local counters or persisted markers best distinguish "awaiting repair" from "repair path exhausted"?

### Snapshot Bootstrap Trust

- Scenario: Startup waits for known-validator snapshot hashes, filters peer snapshot hashes, and then accepts bootstrap state.
- Why outcome is ambiguous: Gossip completeness and same-slot hash conflicts can leave multiple plausible interpretations, and docs still call malicious snapshots a weak area.
- Possible failure semantics:
  - omission: known-validator hashes never observed
  - duplication: multiple peers advertise the same hash
  - reordering: first-seen conflicting hash wins in known-hash construction
  - delay: bootstrap retries while gossip populates
  - partial completion: some known validators observed, others not
  - stale view: stale CRDS entries may influence selection
  - corruption: malicious or inconsistent snapshot material
  - overload: bootstrap timing/port constraints prevent reaching all preferred peers
  - operator error: weak or missing `--known-validator` configuration
- Evidence: `validator/src/bootstrap.rs:858-889`; `validator/src/bootstrap.rs:891-909`; `validator/src/bootstrap.rs:925-1000`; `validator/src/bootstrap.rs:1028-1056`; `SECURITY.md:171-173`.
- Open question: After peer selection, what exact code path verifies the downloaded snapshot archive itself before local restore?

### Incremental Snapshot Cleanliness

- Scenario: Zero-lamport accounts are cleaned after a full snapshot but before an incremental snapshot.
- Why outcome is ambiguous: Full + incremental composition can otherwise reintroduce stale account state on restart.
- Possible failure semantics:
  - omission: needed tombstone-like zero-lamport evidence missing from incremental snapshot
  - duplication: account appears in full snapshot and later in incremental handling
  - reordering: clean runs relative to snapshot timing
  - delay: purge deferred until a later full snapshot
  - partial completion: some zero-lamport accounts filtered, others not
  - stale view: restored bank sees old non-zero data
  - corruption: inconsistent snapshot slot/storage relationships
  - overload: background snapshot backlog may skew ordering
  - operator error: disabling/misconfiguring snapshots
- Evidence: `accounts-db/src/accounts_db.rs:961-988`; `accounts-db/src/accounts_db.rs:2245-2263`; `runtime/src/serde_snapshot.rs:95-110`.
- Open question: Which higher-level invariant or test corpus is treated as authoritative proof that snapshot restore preserves zero-lamport deletions correctly?

### Pruned Local History

- Scenario: Local ledger history is intentionally truncated, while some RPC calls still imply historical access.
- Why outcome is ambiguous: A client may not know whether absence means purge, feature disablement, or genuine nonexistence unless Bigtable fallback is configured.
- Possible failure semantics:
  - omission: requested historical entry no longer local
  - duplication: local and Bigtable histories may overlap
  - reordering: fallback source may be consulted after local miss
  - delay: historical lookup can depend on slower external storage
  - partial completion: local metadata present for some ranges only
  - stale view: external archive may lag or differ operationally
  - corruption: incorrect special-column purge would remove indexes inconsistently
  - overload: history enablement increases disk usage and IOPS
  - operator error: enabling public history without sufficient storage
- Evidence: `ledger/src/blockstore_cleanup_service.rs:47-140`; `ledger/src/blockstore/blockstore_purge.rs:390-430`; `validator/src/commands/run/args/json_rpc_config.rs:72-94`; `docs/src/operations/setup-an-rpc-node.md:57-61`.
- Open question: Which RPC endpoints explicitly distinguish "not retained locally" from "not found anywhere" in the current implementation?

### Failover State Transfer

- Scenario: Operator copies a tower file and flips identity between primary and standby validators.
- Why outcome is ambiguous: File copy, identity switch, and symlink update happen as separate steps across hosts.
- Possible failure semantics:
  - omission: tower not copied
  - duplication: both nodes may transiently believe they can vote
  - reordering: identity switch before tower arrival
  - delay: standby receives tower after active node already unstaked itself
  - partial completion: symlink updated on one host only
  - stale view: standby uses old tower state
  - corruption: copied tower file mismatched with identity
  - overload: restart window closes before switch completes
  - operator error: skipping `--require-tower`
- Evidence: `validator/src/commands/set_identity/mod.rs:43-47`; `docs/src/operations/guides/validator-failover.md:72-103`; `core/src/consensus/tower_storage.rs:153-219`.
- Open question: Beyond presence of the tower file, what code validates that the restored tower is sufficiently fresh for safe failover?

### Plugin Export Success

- Scenario: Validator processes account/runtime changes and notifies loaded plugins.
- Why outcome is ambiguous: Plugin callback success is separate from core validator correctness, and callback errors are logged after the core event already exists.
- Possible failure semantics:
  - omission: plugin disabled or callback skipped
  - duplication: plugin may process the same logical object multiple times across restart/export flows
  - reordering: plugin observes startup restore and live updates with its own ordering concerns
  - delay: plugin or downstream sink may lag
  - partial completion: some plugins succeed, others fail
  - stale view: downstream sink may not reflect latest runtime state
  - corruption: plugin-specific serialization/storage bugs
  - overload: plugin-enabled notification paths add extra work
  - operator error: loading non-conforming plugin config/library
- Evidence: `geyser-plugin-interface/src/geyser_plugin_interface.rs:485-590`; `geyser-plugin-manager/src/geyser_plugin_service.rs:95-121`; `geyser-plugin-manager/src/accounts_update_notifier.rs:136-160`; `SECURITY.md:163-165`.
- Open question: Are any plugin notification failures escalated into validator liveness or startup gating, or are they always non-fatal?

## Map Gaps

- Missing docs: No single operator-facing document explains the exact verification boundary between known-validator snapshot-hash filtering and archive-content verification.
- Missing evidence: I have not yet traced the final archive download/verification path after peer selection, nor the exact RPC handlers that surface pruned-history misses.
- Ambiguous ownership: Snapshot trust is split across docs, bootstrap logic, runtime restore, and security-policy caveats.
- Claims needing deeper code inspection:
  - bootstrap archive verification after peer selection
  - strongest local proof of vote durability
  - exact semantics of RPC "history not retained" responses
  - tower freshness checks during failover beyond file presence
