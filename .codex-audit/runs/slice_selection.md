# Slice Selection Run

## Inputs Read

- `.codex-audit/audit_state.yaml`
- `.codex-audit/repo_profile.md`
- `.codex-audit/promises.md`
- `.codex-audit/system_map.md`
- `.codex-audit/effect_graph.jsonl`
- `.codex-audit/open_questions.md`

## Selection Method

I started from promises rather than modules. Candidate slices were formed only where the map already showed:

- durable state transitions
- cross-boundary side effects
- ambiguous completion or authority confusion
- explicit best-effort / eventual / operator-trust semantics

Each candidate was scored 0-3 on:

- promise_criticality
- state_effect_density
- authority_boundary_risk
- ambiguous_completion_risk
- failure_semantic_richness
- evidence_availability
- audit_tractability
- proof_potential

Total score guided ranking, but not mechanically. Where two candidates scored similarly, I preferred the one that resolves a bigger map contradiction with a tighter worksheet boundary.

## Rejected Broad Targets

| Target | Reason rejected |
| ------ | --------------- |
| Consensus subsystem | Too broad; not a bounded workflow, and would collapse multiple promises and authorities into one target. |
| Runtime / accounts-db | Too broad; better split into nonce lifecycle, snapshot restore, or account-cleaning interactions. |
| RPC subsystem | Too broad; retention/purge/fallback is a tractable slice, but the whole subsystem is not. |
| Bootstrap | Too broad on its own; narrowed to snapshot acceptance and restore. |
| Failover | Too operator-procedure-shaped unless tied to tower continuity and vote durability. |
| Plugins | Entire extension surface is too broad; retained only as a lower-priority bounded export-path candidate. |

## Candidate Slices

| Candidate | Score | Reason |
| --------- | ----: | ------ |
| Bootstrap snapshot acceptance and restore | 22 | High-value trust boundary, durable restart state, strong ambiguous completion, and direct doc/code tension. |
| Vote durability, tower persistence, and failover handoff | 23 | Extremely critical safety promise with durable local state plus operator/network ambiguity. |
| Repair request to repaired-shred acceptance | 22 | Explicit best-effort workflow with remote authority, durable ledger writes, and rich failure semantics. |
| RPC history retention, purge, and fallback semantics | 21 | Strong evidence and tractability; absence semantics are reliability-relevant and operator-visible. |
| Durable nonce lifecycle and replay protection | 18 | Good bounded workflow with strong evidence, but narrower blast radius and weaker proof surface than the top slices. |
| Plugin export notification boundary | 19 | Clear trust boundary and ambiguous downstream completion, but partly operator-trusted/out-of-scope by project policy. |

## Final Ranking

| Rank | Slice | Score | Reason |
| ---- | ----- | ----: | ------ |
| 1 | Bootstrap snapshot acceptance and restore | 22 | Resolves the biggest current trust ambiguity and anchors later restart-related audits. |
| 2 | Vote durability, tower persistence, and failover handoff | 23 | Slightly higher raw score, but better second because proving durable vote completion likely fans into broader replay observation logic. |
| 3 | Repair request to repaired-shred acceptance | 22 | Best bounded degraded-mode workflow after bootstrap and vote continuity. |
| 4 | RPC history retention, purge, and fallback semantics | 21 | High operator value and strong evidence, but lower safety criticality than the top three. |
| 5 | Durable nonce lifecycle and replay protection | 18 | Good bounded runtime workflow, but less central than bootstrap / voting / repair. |

## Recommended First Slice

`bootstrap-snapshot-acceptance`

This is first because:

- it is the clearest contradiction already visible in the map
- it combines durable state, remote trust, operator config, and restart safety
- it is bounded enough to worksheet concretely without first auditing all of consensus
- later slices about failover, replay, or RPC all assume the node started from acceptable local state

## Notes for Next Phase

- Keep the first worksheet narrow: known-validator snapshot-hash discovery, peer filtering, archive acceptance boundary, and local restore entrypoints.
- Do not drift into all snapshot internals immediately; prove the acceptance path first.
- If archive verification fans into a separate bounded workflow, split it during worksheet rather than broadening the initial slice.
