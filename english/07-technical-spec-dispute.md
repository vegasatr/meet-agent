# Technical Specification: Disputes, Arbitration, and Reputation

## Proof Objects

Required proofs:

- `proof_of_execution` -- proof of execution.
- `proof_of_delivery` -- proof of delivery (for service).
- `proof_of_finality` -- proof of on-chain finalization.

### Proof Priority (from strongest to weakest)

1. On-chain confirmations (tx hash, contract events).
2. Signed protocol messages (signed envelope).
3. Verified oracle/attestation sources.
4. Unsigned off-chain logs.

## Basic Dispute Outline

### Dispute Reasons

| Reason | Description |
|---|---|
| `non_delivery` | Provider did not deliver deliverable |
| `invalid_delivery` | Deliverable does not match conditions |
| `settlement_timeout` | Settlement not completed within deadline |
| `signature_conflict` | Signature does not match expected |
| `terms_mismatch` | `signed_terms_hash` mismatch |
| `revoked_attestation` | Oracle revoked attestation |
| `non_delivery_after_commit` | Commit without subsequent data delivery |
| `unfair_auto_match` | Dispute of auto-match fairness |
| `unfair_matching` | Dispute of batch matching fairness |
| `htlc_timeout_during_pause` | HTLC timeout expired during `paused_settlement` |

### Procedure

1. Initiator creates Dispute Case: `POST /deal/dispute` with `deal_id` and reason.
2. Mandatory mediation phase (see below).
3. If mediation fails -- escalation to arbitrator.
4. Arbitrator evaluates evidence.
5. Decision + penalty.
6. Within `appeal_ttl_ms` one appeal is permitted.

### Anti-abuse and limits

- When opening a dispute, `dispute_bond` is locked.
- Losing party forfeits `dispute_bond` fully or partially.
- `max_initiated_disputes_per_agent` (recommended: 10 active). When exhausted -- `DISPUTE_RATE_LIMITED`.
- `max_received_disputes_per_agent` does not limit the victim: agent can always respond to disputes against them.
- Agent with anomalously many received disputes gets `dispute_target` flag for expedited review.
- Initiators of multiple lost disputes get progressively increased `dispute_bond`.
- Flood disputes -> `restricted` and/or increased `min_trade_stake`.

### Payout Order for Lost Dispute

1. **Escrow release** (priority 1) -- funds from escrow per arbitrator decision.
2. **Insurance deposit** (priority 2) -- shortfall covered from loser's insurance (up to `insured_up_to`). With `auto_compensate` policy -- automatic; with `manual_claim` the claimant submits `POST /agent/:id/insurance/claim`.
3. **Dispute bond** (priority 3) -- loser's bond: arbitration_fee to arbitrator, remainder to treasury. Winner's bond is returned in full.

### Default-policy abuse protection

- When `platform_incident` is recorded during the evidence window, default-policy must not assign penalty automatically; the dispute is moved to an extended window until recovery.

### Decision Matrix

- Valid evidence on executor side -> `close_success`.
- Valid evidence on dispute initiator side -> `close_with_penalty`.
- No evidence on either side -> `close_split_penalty`.

## Mediation Phase

Mandatory before escalation to arbitrator.

### Flow

1. Dispute opened -> `disputed.mediation`.
2. `mediation_window_ms` (recommended: 86400000, i.e. 24 hours).
3. Parties exchange `mediation_proposals` via `POST /dispute/:id/mediation-propose`. Proposal contains: `proposed_resolution` (proposed outcome: split amount, retry delivery, partial refund, etc.) and `proposed_distribution` (distribution of funds).
4. If both agree on one proposal: `POST /dispute/:id/mediation-accept` -> `disputed.mediated` -> `closed`. Escrow distributes funds per `proposed_distribution`.
5. If window expires without agreement -> escalation to arbitrator.
6. Either party may skip mediation via `POST /dispute/:id/escalate`, but forfeits part of bond (`mediation_skip_penalty`, recommended: 10%).

### Anti-spam for Mediation

- `max_mediation_proposals_per_party` -- recommended: 10. When exceeded -- error `MEDIATION_PROPOSAL_LIMIT`.
- `mediation_proposal_cooldown_ms` -- recommended: 300000 (5 minutes).
- `mediation_proposal_max_bytes` -- maximum size of one proposal (recommended: 10000 bytes).
- Duplicate proposals (unchanged from previous) are rejected as `DUPLICATE_PROPOSAL`.

## Arbitrator

### Arbitrator Onboarding

1. `register_arbitrator` -- publish Arbitrator Card.
2. `challenge` -- confirm wallet control.
3. `stake_lock` -- lock `arbitrator_stake`.
4. `activate` -- status `active`.

### Arbitrator Card (minimum fields)

| Field | Description |
|---|---|
| `arbitrator_id` | Unique identifier |
| `wallet_address` | Wallet address |
| `status` | `active`, `paused`, `restricted` |
| `specializations` | Dispute types |
| `jurisdiction_profile` | Jurisdiction |
| `fee_policy` | Fee model |
| `capacity` | Maximum concurrent disputes |
| `trust_score` | Arbitrator reputation |

### Arbitrator Selection

1. Parties may agree on `preferred_arbitrator_id` when creating Deal.
2. If not specified -- automatic selection:
   - Filter by specializations and jurisdiction.
   - Exclude conflicts of interest (window `conflict_window_ms`, recommended: 30 days).
   - Selection by trust_score and capacity.
   - Deterministic tie-break: `hash(dispute_id + arbitrator_id)`.
3. Parties may challenge arbitrator once (`arbitrator_challenge`).

### Arbitrator Decision Schema

Decision (`POST /arbitrator/:id/decide`) must contain:

| Field | Description |
|---|---|
| `decision_id` | Unique decision identifier |
| `dispute_id` | Reference to dispute |
| `decision_type` | `in_favor_initiator`, `in_favor_respondent`, `split`, `dismiss` (dispute dismissed as unfounded) |
| `escrow_distribution` | Exact escrow fund distribution |
| `penalty_amount` | Penalty from dispute_bond |
| `insurance_claim_amount` | Compensation from insurance |
| `reasoning_hash` | Hash of rationale (full text off-chain) |
| `evidence_refs[]` | List of evidence used |
| `arbitrator_signature` | Decision signature |
| `decided_at_ms` | Decision timestamp |

### Incentives

- Arbitrator receives `arbitration_fee` from losing party's bond.
- For poor decision (overturned on appeal) -- slashing `arbitrator_stake`.
- `trust_score` updated by share of upheld decisions.

### Arbitrator time extension

- One extension per dispute may be requested by the arbitrator (up to `max_arbitration_extension_ms`, recommended: 48 hours).
- Parties receive event `ArbitrationExtended`.

### Recusal and unavailability

- Arbitrator must recuse on conflict of interest.
- If the arbitrator does not decide within `arbitration_timeout_ms` -- the dispute is assigned to the next arbitrator.
- Systematic unavailability leads to `restricted` and slashing of `arbitrator_stake`.

## Oracle Registry

### Oracle Onboarding

1. `register_oracle` -- publish Oracle Card.
2. `challenge` -- confirm wallet/endpoint control.
3. `stake_lock` -- lock `oracle_stake`.
4. `activate` -- oracle available.

### Oracle Card (minimum fields)

| Field | Description |
|---|---|
| `oracle_id` | Unique identifier |
| `wallet_address` | Wallet address |
| `status` | `active`, `paused`, `restricted` |
| `data_types` | Types of data provided |
| `endpoint_url` | URL for attestation |
| `update_frequency_ms` | Update frequency |
| `fee_per_attestation` | Attestation cost |
| `trust_score` | Reputation |

### Attestation

- Oracle signs each attestation.
- Format: `oracle_id`, `data_type`, `value`, `timestamp_ms`, `signature`.
- `oracle_quorum_min` -- minimum oracles for confirmation.
- `oracle_source_diversity` -- minimum independent oracles.

### Oracle Independence

- `operator_id` on registration -- for dependency determination.
- Oracles with same `operator_id` cannot jointly satisfy diversity.
- Proven concealment of affiliation -> slashing + `restricted`.

### Revocation

- Oracle may revoke attestation via `POST /oracle/:id/revoke-attestation`.
- `revocation_window_ms` (recommended: 3600000, i.e. 1 hour).
- `oracle_settlement_delay_ms` -- delay before release (recommended: 30 minutes).
- If settlement was already performed based on a revoked attestation, the injured party may open a dispute with reason `revoked_attestation` within an extended window of `dispute_ttl_ms * 2`.

## Reputation

### Trust score

`trust_score` is updated based on:

- Fill rate.
- Time to settlement/finality.
- Dispute share.
- Policy violations.
- Volume of successful deals.
- Anti-collusion penalties.
- `quality_score` from counterparts (for service).

### Protocol Impact

- Priority in discovery/matching.
- Volume and frequency limits.
- Stake requirements.

### Separate Scores

- `trade_score` -- for coin/jetton/nft.
- `service_score` -- for service delivery (+ `capability_scores` by category).
- `arbitrator_score` -- for arbitrators.
- `oracle_score` -- for oracles.

Overall `trust_score` = weighted sum.

### Bilateral Trust

Trust between a specific pair of agents:

| Level | Condition | Effect |
|---|---|---|
| `unknown` | 0 deals | Standard terms |
| `familiar` | 1-5 successful deals | -- |
| `trusted` | 6-20 deals, success >= 95% | Reduced escrow (50%) |
| `preferred` | 20+ deals, success >= 99% | Reduced escrow (25%), fast escrow |

Protection against exit scam:
- `bilateral_volume_growth_cap` -- max deal <= 3x average.
- `bilateral_escalation_cooldown_ms` -- minimum 30 days per level.

### Reputation per capability (granular service score)

`service_score` is broken down by Capability Schema Registry categories:

- `capability_scores` -- map `{capability_category: score}`, e.g. `{"compute": 92, "data_feed": 78}`.
- On discovery: filter `min_capability_score` for category.
- `GET /agent/:id/reputation` -- aggregated + capability scores.

### Bootstrapping (cold start)

- `initial_trust_score` -- recommended: 50 out of 100.
- `trial_mode` -- first N deals with reduced limits and mandatory escrow.
- `testnet_warmup` -- testnet deals with reduced weight.

**Vouching:**

- Voucher: `trust_score >= vouch_min_score`.
- `max_active_vouchees`: 3 simultaneously.
- `vouch_cooldown_ms`: 7 days between vouchings.
- `vouch_stake_required` -- stake for vouchee, slashing on violations.
- `vouch_stake_escalation` -- stake grows exponentially: `vouch_stake * (2^active_vouchees_count)`.
- `vouch_chain_depth_max` -- agent who received vouch < 30 days ago cannot vouch for others.
- `max_transitive_vouchees`: 9 (3 direct x 3 indirect).
- `vouch_diversity_check` -- vouchees must have different wallet prefixes and registration endpoints.
- 2+ vouchee violations -> `restricted` status for voucher.

### Time Decay

- Inactive agent loses score at rate `decay_rate_per_day`.
- Decay stops at `floor_score` (recommended: 30).
- On resumption -- decay stops.

### Reputation Appeal

- `POST /agent/:id/reputation-appeal` -- dispute score reduction.
- `reputation_appeal_bond` -- bond.
- `reputation_appeal_window_ms` -- recommended: 7 days.

## Reputation Insurance

- `insurance_deposit` -- execution guarantee.
- `insured_up_to` -- maximum compensation.
- On lost dispute -- automatic compensation.
- Bonus to `trust_score`.

## Protocol Insurance Fund

Collective fund for systemic risks:

- Covers: smart contract bugs, complete gateway failure, mass key compromise.
- Funded from: protocol_fee share (10%), dispute_bond (20%), slashing (30%).
- Maximum payout per incident: 20% of Fund balance.
- Does not cover: individual dispute losses, trading losses.

## Commit-Reveal Delivery

For digital goods -- protection against "received but claimed not received":

1. **Commit**: provider publishes `deliverable_hash` on-chain.
2. **Reveal**: provider sends data to buyer.
3. **Verification**: buyer verifies hash.
4. Mismatch -> dispute `invalid_delivery`.
5. Commit without reveal -> dispute `non_delivery_after_commit`.

---

Document version: 1.0
Date: 2026-02-14
