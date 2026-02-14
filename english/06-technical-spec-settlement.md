# Technical Specification: Settlement (Escrow, HTLC, Fees)

## Canonical Terms Hash (`signed_terms_hash`)

The hash is computed from the canonical representation of the following Deal fields in strict order:

1. `deal_id`
2. `participants` (sorted by `agent_id` lexicographically)
3. `legs` (sorted by `leg_index`)
4. `settlement_mode`
5. `expiry_ms`
6. `protocol_fee`
7. `gateway_fee`
8. `bundle_atomicity` (if bundle; otherwise `null`)
9. `rollback_policy` (if bundle; otherwise `null`)
10. `bundle_timeout_ms` (if bundle; otherwise `null`)
11. `preferred_arbitrator_id` (if specified; otherwise `null`)
12. `oracle_ids` (sorted lexicographically)
13. `oracle_quorum_min` (if used; otherwise `null`)
14. `escrow_timeout_ms` (if escrow/htlc; otherwise `null`)
15. `conditions` (canonical array)
16. `gas_split_policy` -- for standard and exchange deals: who pays gas for on-chain contract creation (`initiator_pays` | `split_equal` | `receiver_pays`, default `initiator_pays`); for bundle -- distribution of gas costs among participants per Deal terms.
17. `funding_timeout_ms` (if escrow; otherwise `null`)
18. `counterparty_funding_timeout_ms` (if bilateral escrow; otherwise `null`)
19. `nft_royalty` (canonical array)
20. `early_termination_fee` (if subscription; otherwise `null`)
21. `min_subscription_periods` (if subscription; otherwise `null`)
22. `rollback_gas_policy` (if bundle; otherwise `null`)

### Rules

- `null` fields are included as explicit `null` (not omitted).
- Format: `canonical_payload_format` from crypto_profile, then `hash_function_ref`.
- Both parties must independently compute the hash and verify before settlement.
- Mismatch blocks Deal transition to `settling` (error `TERMS_HASH_MISMATCH`).

## ValueLeg (unified structure)

Each leg of a deal is described by:

| Field | Description |
|---|---|
| `asset_type` | Asset type (coin, jetton, nft, service, bundle) |
| `asset_id` | Asset identifier |
| `amount_or_units` | Amount or quantity |
| `owner_agent_id` | Current owner |
| `receiver_agent_id` | Receiver |
| `validation_rule` | Validation rule |

For `service` the following are additionally required:

- `deliverable_schema_ref` -- reference to deliverable schema.
- `acceptance_rule` -- acceptance rule (DSL or custom).
- `sla_seconds` -- SLA for execution.
- `evidence_type` -- type of delivery proof.

## Settlement Adapter (abstract interface)

Each settlement mode is implemented via a Settlement Adapter:

| Method | Description |
|---|---|
| `initiate(deal)` | Initialize settlement |
| `verify_deposit(tx_hash)` | Verify deposit on-chain |
| `verify_release(tx_hash)` | Verify release on-chain |
| `get_status(deal_id)` | Current status |
| `estimate_gas(deal)` | Estimate gas costs |

Core MVP adapters: `direct_transfer`, `htlc_swap`, `escrow`, `milestone_settlement`.
Extension: `bridge_swap`. Modifier `partial_fill` is implemented on top of `escrow`/`htlc_swap`.

## Deal Cancellation Reason Taxonomy

Required field `cancel_reason` in `DealCancelled`:

| Reason | Description |
|---|---|
| `mutual_cancel` | Both parties agreed to cancel |
| `subscription_cancel` | Subscription cancellation |
| `funding_timeout` | Deposit not made in time |
| `terms_verification_timeout` | Hash not confirmed in time |
| `auto_match_rejected` | One party rejected auto-match |
| `pipeline_abort` | Cancellation within pipeline |
| `template_budget_exceeded` | Template budget exhausted |
| `circuit_breaker_triggered` | Template circuit breaker triggered |
| `session_rotation_limit` | Session rotation limit exhausted |
| `settlement_start_timeout` | Agent did not start settlement |
| `counterparty_offline_timeout` | Counterparty offline too long |
| `chain_downtime_cancel` | Cancellation due to prolonged chain downtime |
| `delegation_revoked` | Delegation revoked |
| `gateway_deprecation` | Gateway ceasing operation |
| `agent_draining` | Agent in draining mode |
| `htlc_timeout_during_pause` | HTLC timeout during paused_settlement |

## Settlement Modes

### Direct Transfer

Direct transfer between wallets. Allowed only for one-way transfers (payment, donation). Prohibited for service and bilateral exchanges.

**Flow:**
1. Payer invokes `POST /deal/settle` with `action: transfer` and tx hash.
2. Gateway verifies on-chain transfer.
3. Deal -> `settling` -> `settled_pending_finality` after `min_confirmations`.

### HTLC Swap

Atomic swap via Hash Time-Locked Contract for bilateral exchanges of coin/jetton <-> coin/jetton.

**Lifecycle:**
```
created -> initiator_locked -> both_locked -> released (or refunded)
```

**Flow:**
1. Initiator generates `secret` and `secret_hash = hash(secret)`.
2. Initiator creates HTLC, locks funds.
3. Counterparty creates mirror HTLC with the same `secret_hash`.
4. Initiator reveals `secret` to receive counterparty funds.
5. Counterparty uses `secret` to receive initiator funds.
6. If `secret` is not revealed before `htlc_timeout_ms` -- refund to both parties.

**Required HTLC fields:**

| Field | Description |
|---|---|
| `htlc_id` | Identifier |
| `deal_id` | Reference to Deal |
| `secret_hash` | Secret hash |
| `initiator_agent_id` / `counterparty_agent_id` | Parties |
| `initiator_asset` / `counterparty_asset` | Assets (type, id, amount) |
| `htlc_timeout_ms` | Timeout for reveal |
| `counterparty_lock_timeout_ms` | Timeout for counterparty (< `htlc_timeout_ms`) |

**Rules:**
- Counterparty HTLC timeout strictly less than initiator HTLC.
- Minimum difference: `htlc_timeout_safety_margin_ms` (recommended: 300000).
- Secret: minimum 256 bits, cryptographically secure random.

### Escrow

On-chain TON smart contract. Gateway and participants do not have custody.

**Lifecycle:**
```
created -> funded -> active -> releasing -> released (or refunded)
```

**Escrow types:**
- `unilateral` -- one party deposits funds.
- `bilateral` -- both parties deposit funds.

**Required fields:**

| Field | Description |
|---|---|
| `escrow_id` | Identifier |
| `deal_id` | Reference to Deal |
| `escrow_type` | `unilateral` or `bilateral` |
| `release_conditions` | Release conditions |
| `refund_conditions` | Refund conditions |
| `timeout_ms` | Timeout for automatic refund |
| `funding_timeout_ms` | Time for deposit (recommended: 600000) |
| `counterparty_funding_timeout_ms` | For bilateral (recommended: 300000) |
| `protocol_fee_amount` | Fixed protocol fee amount |
| `gateway_fee_amount` | Fixed gateway fee amount |

**Release rules:**
- `auto_release` -- upon receiving proof.
- `manual_release` -- by signature of both parties.
- `arbiter_release` -- by arbitrator decision (upon dispute).

**Refund rules:**
- `timeout_refund` -- automatic upon `timeout_ms` expiration.
- `mutual_refund` -- both parties signed cancel.
- `arbiter_refund` -- by arbitrator decision.

**Funding rules for bilateral escrow:**
1. Total `funding_timeout_ms` from `created`.
2. After first party deposit -- second party must deposit within `counterparty_funding_timeout_ms`.
3. If second party did not deposit -- refund first party, escrow -> `cancelled`, Deal -> `failed`.

**Contract interface (minimal):**
- `deposit(deal_id, amount)`
- `release(deal_id, proof_hash, signature)`
- `partial_release(deal_id, milestone_id, amount, proof_hash, signature)`
- `refund(deal_id, reason, signature)`
- `dispute_release(deal_id, arbiter_decision_hash, arbiter_signature)`
- `get_status(deal_id)`
- `update_key(deal_id, new_pubkey, old_key_signature)`
- `freeze(deal_id, reason)` / `unfreeze(deal_id, authorization_signature)`
- `amend_terms(deal_id, new_terms_hash, amendment_id, parties_signatures[])`
- `partial_refund(deal_id, amount, beneficiary, reason, signature)` -- partial refund (amendment with reduced amount, partial compensation).
- `commit_deliverable(deal_id, deliverable_hash, provider_signature)` -- commit deliverable hash for Commit-Reveal Delivery.

**Emergency escrow recovery:** when a contract bug blocks release/refund, emergency recovery is possible after `emergency_timeout_ms` (recommended: 30 days after last state change). Requires multi-sig from `emergency_recovery_threshold` of `emergency_recovery_signers[]` (recommended: 3-of-5). Signer composition: max 1 affiliated with gateway operator, min 2 independent (auditors, governance, community). For deals with amount above `emergency_independent_signer_threshold` (recommended: 1000 TON) independent signers are mandatory. Signer list is published on-chain and available via `GET /deal/:id/escrow`. Events: `EmergencyRecoveryInitiated`, `EmergencyRecoveryCompleted`.

### Milestone Settlement

Phased settlement for service. Escrow with milestone schedule.

**Flow:**
1. Gateway creates Escrow with milestone schedule.
2. Payer deposits first milestone amount (or full amount).
3. After milestone execution, provider invokes `POST /deal/proof`.
4. Gateway/oracle verifies proof, initiates partial release.

### HTLC: behavior during paused_settlement

During `paused_settlement` HTLC timers on-chain are NOT paused (unlike escrow). Mandatory HTLC Emergency Reveal:

1. During `paused_settlement` gateway checks time remaining until `htlc_timeout_ms`.
2. If remaining < `htlc_emergency_reveal_threshold_ms` (recommended: 25% of timeout, minimum 600000) -- event `HTLCEmergencyRevealRequired` with `deadline_ms`.
3. Initiator must perform on-chain reveal before deadline.
4. If reveal not performed -> `refunded`, Deal -> `disputed` (`htlc_timeout_during_pause`).
5. On `key_compromise_reported`: initiator must reveal secret with the new key (on-chain reveal is independent of off-chain key). If secret is lost, HTLC Emergency Reveal flow applies; if not revealed before timeout, HTLC moves to `refunded`, Deal -> `disputed`.
6. Recommendation: for deals with high risk of `paused_settlement` use `escrow` instead of HTLC.

### Partial Fill

Partial execution with atomic confirmation of each sub-fill.

- Each sub-fill has `sub_fill_id`, `amount`, `proof_hash`.
- `min_fill_amount` is set by profile policy.
- Prohibited for `service` and `nft`.
- Unfilled remainder is returned when Deal closes.

**Partial fill with escrow:**

1. Both parties deposit full amount into bilateral escrow.
2. On each sub-fill -- `partial_release` on-chain (proportional to amount).
3. Remainder -- refund when Deal closes.

**Partial fill with htlc_swap:**

1. For each sub-fill -- a separate mini-HTLC.
2. Economic check: `estimated_total_gas = sub_fill_count * 2 * estimated_htlc_gas`. If > `deal_amount * 0.05` -- rejection with `HTLC_GAS_EXCEEDS_VALUE`, suggest escrow.
3. `max_sub_fills` -- recommended: 20.

**Partial fill with direct_transfer:** prohibited.

## Settlement Atomicity Matrix

| Scenario | Allowed settlement mode |
|---|---|
| One-way transfer (payment) | `direct_transfer`, `escrow` |
| Bilateral coin/jetton exchange | `htlc_swap`, `escrow` |
| NFT deal | `escrow` (partial fill prohibited) |
| Service deal | `escrow`, `milestone_settlement` |
| Bundle `all_or_nothing` | `escrow` |
| Bundle `best_effort` | `htlc_swap` for coin/jetton legs |
| Cross-chain | `bridge_swap` (escrow on TON side) |

`direct_transfer` for bilateral exchanges is **prohibited** at any level of bilateral trust. Profile policy may extend allowed modes but must not relax baseline prohibitions for service and bilateral exchanges.

## Settlement Initiation

### For escrow

1. Gateway creates on-chain Escrow Contract.
2. Event `SettlementInitiated` to both parties.
3. Each party invokes `POST /deal/settle` with `action: deposit` and tx hash.
4. Gateway verifies on-chain deposit.
5. All deposits confirmed -- Deal -> `settling`.

### For HTLC

1. Gateway creates on-chain HTLC Contract.
2. Event `SettlementInitiated` to both parties.
3. Initiator makes deposit. Deal -> `settling`.
4. Counterparty creates mirror HTLC.
5. Standard reveal flow.

### General rules

- `POST /deal/settle` is idempotent.
- `settlement_start_timeout_ms` (recommended: 600000) -- if expired without action, Deal -> `failed`.

## Settlement and Finality Parameters

| Parameter | Description | Recommended |
|---|---|---|
| `min_confirmations` | Minimum on-chain confirmations | Network dependent |
| `settlement_timeout_ms` | Settlement timeout | Profile policy |
| `finality_timeout_ms` | Finality timeout | Profile policy |
| `pause_timeout_ms` | Maximum in `paused_settlement` | Profile policy |
| `arbitration_timeout_ms` | Arbitrator decision timeout | 86400000 (24h) |
| `funding_timeout_ms` | Time for deposit | 600000 (10min) |
| `terms_verification_timeout_ms` | Time for confirm-terms | 120000-600000 |
| `acceptance_window_ms` | Time for deliverable verification | By asset type |
| `dispute_ttl_ms` | Window for publishing evidence | Profile policy |
| `appeal_ttl_ms` | Appeal window | Profile policy |
| `rollback_timeout_ms` | Maximum time for rollback of bundle legs | Profile policy |
| `settlement_start_timeout_ms` | Time from accepted to first settle action | 600000 (10min) |
| `offline_deal_grace_ms` | Grace period before dispute on offline | 300000 (5min) |
| `offline_critical_threshold_ms` | Threshold for automatic paused_settlement | 1800000 (30min) |

## Fee Model

### Structure

- `protocol_fee` -- to protocol treasury.
- `gateway_fee` -- to gateway operator.
- `arbitrator_fee` -- to arbitrator (only upon dispute).
- `oracle_fee` -- to oracle for attestation.

### Fee modes

- `none` | `fixed` | `bps` (basis points).
- Fees are declared before deal acceptance.
- Final cost is unambiguously computable from `signed_terms_hash`.

### Collection mechanism

**Escrow:** fee deducted on release (not on deposit). On release escrow atomically in a single on-chain transaction: beneficiary receives `amount - protocol_fee - gateway_fee`, treasury receives `protocol_fee`, gateway -- `gateway_fee`. On refund -- fee not charged. On dispute_release -- fee deducted from the losing party's amount.

**HTLC:** fee deducted from each party's amount on release. On refund -- fee not charged.

**Direct transfer:** fee-first model. Order:
1. Payer sends fee to treasury and gateway.
2. Gateway verifies fee on-chain.
3. Only after confirmation -- main transfer.
4. `fee_payment_timeout_ms` (recommended: 300000). Not paid -- `failed`.
5. Fee paid but transfer not executed -- fee is NOT refunded (anti-abuse).
6. Alternative: `DirectTransferWithFee` contract (fee + transfer atomically).

**Milestone:** fee distributed proportionally across milestones. Partial execution -- fee only for completed milestones.

### Stake economy

| Stake type | Purpose |
|---|---|
| `agent_stake` | Access to limits |
| `arbitrator_stake` | Guarantee of arbitrator integrity |
| `oracle_stake` | Guarantee of data reliability |
| `dispute_bond` | Deposit when opening a dispute |

All stake and bond -- on-chain. Slashing rules are published in profile policy, not changed retroactively.

## Deal Amendment

Changing terms of active Deal:

1. Initiator: `POST /deal/:id/amend`.
2. All parties: `POST /deal/:id/amend-accept`.
3. Gateway recalculates `signed_terms_hash`.
4. On escrow: `amend_terms()` on-chain.
5. `max_amendments_per_deal` -- recommended: 5.

Restrictions:
- Amendment from `settling` only for non-amount fields.
- For HTLC: amendment prohibited after contract creation.

## NFT (specifics)

- `partial_fill` prohibited (NFT indivisible).
- Before Deal creation gateway must verify NFT ownership on-chain (`nft_ownership_check`). If ownership is not confirmed, Quote is rejected with error `NFT_OWNERSHIP_UNVERIFIED`.
- `escrow` settlement mode required for NFT legs.
- For NFTs with royalties (TON NFT standard): `royalty_amount` and `royalty_address` are read from on-chain NFT metadata; royalty is automatically deducted on escrow release and sent to `royalty_address`; royalty is included in Deal terms and in `signed_terms_hash` via field `nft_royalty` in legs.
- NFT transfer via escrow: (1) Seller deposits NFT into escrow; (2) Buyer deposits payment (coin/jetton); (3) On release: NFT to buyer, payment (minus royalty and fees) to seller, royalty to `royalty_address`.
- `nft_metadata_snapshot` -- gateway fixes NFT metadata snapshot at Deal creation time for dispute resolution.

## Bundle

Multiple ValueLegs in one deal.

### Atomicity modes

- `all_or_nothing` -- all legs atomically; on failure of one -- rollback all.
- `best_effort` -- each leg independent; partial execution allowed.
- `sequential` -- in order of `leg_index`; failure at step N stops the chain.

### Multi-party bundles

- Bundle may include 3+ agents (A->B, B->C, C->A). For circular bundles `escrow` is mandatory.
- All participants sign the same `signed_terms_hash` before settlement.

### Bundle escrow architecture

- A separate Escrow Contract per leg (`escrow_per_leg`). All escrows are tied to one `deal_id` and `bundle_id`.
- Coordination via on-chain Bundle Coordinator Contract:
  1. Each leg-escrow notifies Coordinator of deposit (`leg_funded`).
  2. When all legs are `funded` -> Coordinator allows transition to `active`.
  3. Release of all legs is initiated by Coordinator atomically (or per atomicity rules).
- `all_or_nothing`: Coordinator performs release of all legs in one transaction; if one leg cannot be released -- rollback all.
- `sequential`: Coordinator allows release in `leg_index` order; failure at step N stops the chain and rollback legs 0..N-1.
- `best_effort`: each leg-escrow may be released independently.
- Gas costs for Coordinator and leg-escrow are distributed per `gas_split_policy` in Deal.

### Rollback and guarantees

- `direct_transfer` for any leg in an `all_or_nothing` bundle is **forbidden** (does not support rollback).
- `all_or_nothing` with non-escrow legs: gateway must warn participants with event `BundleAtomicityWarning`.
- When creating a bundle each participant deposits `rollback_gas_reserve` in escrow in addition to the main amount. On successful completion `rollback_gas_reserve` is returned; on rollback it is used to pay gas.
- Rollback must complete within `rollback_timeout_ms`. If rollback is impossible (on-chain finality passed) -- bundle moves to `disputed`.

### Bundle Coordinator Contract (summary)

- Gas costs for rollback per `rollback_gas_policy` (`initiator_pays` | `proportional` | `shared_equal`). If failure initiator cannot be determined -- default `proportional`.

## Cross-chain Settlement Mode (bridge_swap)

For deals with assets outside TON.

### Mechanics

1. TON-side locks funds in escrow on TON.
2. External-side locks funds in HTLC/escrow on external chain.
3. Bridge oracle confirms lock on external chain (signed attestation).
4. Upon confirmation of both locks -- release via cross-chain HTLC (shared secret).
5. On timeout -- refund on both sides.

### Bridge oracle requirements

- `data_type: bridge_attestation` in Oracle Registry.
- `bridge_oracle_quorum_min` -- minimum oracles for confirmation (recommended: 3).
- Real-time monitoring of external chain.

Recommended `bridge_attestation_timeout_ms`:

| Chain | Timeout | Reason |
|---|---|---|
| Ethereum | 900000 (15 min) | ~64 slots finality |
| Solana | 30000 (30 sec) | Fast finality |
| Tron | 180000 (3 min) | |
| BNB Chain | 180000 (3 min) | |
| Unlisted | `estimated_finality_ms` at registration | |

### Restrictions

- Always requires escrow on TON-side.
- Gas on external chain is paid by the party with the external wallet.
- Protocol does not manage external chain contracts -- only verifies via bridge oracle.

## Smart Contract Upgrade

On-chain contracts are immutable after deployment. Updates via versioned deployment:

1. Gateway deploys new version.
2. Event `ContractVersionUpdated` with `contract_type`, `old_version`, `new_version`, `new_contract_address`, `migration_deadline_ms`.
3. Agents verify contract (source code in TON explorer). Gateway provides `contract_diff` -- description of changes.
4. Until `migration_deadline_ms` -- both versions. Agents with `compliance_mode: regulated` get extended deadline (x2).
5. After deadline new deals only on new version. Old contracts keep running until associated deals close.
6. On critical bug gateway may declare `emergency_migration` (minimum 24 hours). Event `EmergencyContractMigration`.

## Priority Settlement Queue

During recovery after chain downtime, settlement is processed with priority.

### Priority formula

```
priority_score = urgency_weight * time_to_expiry_inverse
               + value_weight * deal_value_normalized
               + trust_weight * avg_trust_score
```

| Weight | Recommended | Description |
|---|---|---|
| `urgency_weight` | 0.5 | Deals close to `settlement_timeout_ms` -- highest priority |
| `value_weight` | 0.3 | High-value deals -- elevated priority |
| `trust_weight` | 0.2 | High-trust agents -- bonus |

### Fairness guarantee

- No deal can wait longer than `max_queue_wait_ms` (recommended: 300000). Upon reaching -- maximum priority.
- Formula is published in profile policy, not changed retroactively.

## HTLC Secret Management

### Mandatory requirements

- Secret: cryptographically secure random, minimum 256 bits.
- Storage: encrypted storage on initiator side.
- Secret is NOT transmitted through gateway before reveal.

### Recommended practices

- **Encrypted backup**: secret encrypted with agent's key, backup storage.
- **Split-secret (Shamir)**: for high-value deals -- N shares, threshold K (recommended: 2-of-3).
- **Recovery address**: when creating HTLC -- `recovery_address` (if secret lost + timeout, funds -> recovery_address instead of refund to counterparty). Initiator-side only.
- **Secret reveal monitoring**: monitoring on-chain reveal events (counterparty reveal -> initiator sees secret on-chain and can claim).

---

Document version: 1.0
Date: 2026-02-14
