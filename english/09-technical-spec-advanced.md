# Technical Specification: Advanced (Templates, Groups, Pipelines, Capabilities, Auto-negotiation)

## Exchange Deal Type

First-class type for bilateral exchange (A gives X, B gives Y).

### Difference from Bundle

- `exchange` -- exactly 2 legs, 2 participants. Simplified flow.
- Bundle -- 2+ legs, 2+ participants. Coordinator Contract.

### Structure

- `deal_type`: `exchange`
- `leg_give` -- what initiator gives.
- `leg_receive` -- what initiator receives.
- `settlement_mode`: `htlc_swap` (coin/jetton) or `escrow` (with NFT).

Restrictions: service is NOT allowed in exchange.

### Mapping to Canonical Terms Hash

`leg_give` -> `legs[0]`, `leg_receive` -> `legs[1]`, standard processing.

## Auto-negotiation

### Auto-accept policy

In Agent Card `auto_negotiation_policy`:

- `auto_accept_rules`:
  - `max_price` / `min_price`
  - `min_counterparty_trust_score`
  - `accepted_asset_types`
  - `max_amount`, `max_daily_volume`
  - `required_settlement_mode`

- `auto_counter_rules`:
  - `price_adjustment_strategy`: `fixed_spread`, `percentage`, `oracle_based`
  - `max_auto_counter_rounds` (recommended: 3)
  - `auto_counter_cooldown_ms` (recommended: 5000)

- `blacklist` / `whitelist`

### Preference descriptor (negotiation_preferences)

For Intent the agent may specify `negotiation_preferences`:

- `reservation_price` -- minimum or maximum acceptable price.
- `urgency` -- `low` | `medium` | `high` (affects spread willingness).
- `preferred_settlement_modes` -- list of preferred settlement modes in priority order.
- `preferred_counterparty_criteria` -- criteria for selecting counterparty.

### Automated matching

1. Intent with auto_accept enters fast-match pool.
2. Gateway searches for counterparts with compatible rules.
3. When compatible -- Deal in `auto_matched_pending`.
4. Both parties confirm via `POST /deal/:id/confirm-terms` within `auto_match_confirm_window_ms` (recommended: 30 seconds).
5. If both confirmed -- `accepted`.
6. If not -- `cancelled` without penalty.

Auto-match restrictions:
- Prohibited for `service` and `bundle`.
- `auto_match_max_amount` set by profile policy.
- `auto_match_cooldown_ms` (recommended: 1000).

Gateway must issue `auto_match_receipt` -- signed document with selection justification (list of considered candidates, scores, reason for choice). Challenge: `POST /deal/dispute` with reason `unfair_auto_match` within `dispute_ttl_ms`.

## Deal Templates

### Structure

- `template_id`, `creator_agent_id`
- `counterparty_agent_id` (optional)
- `deal_type`: `standard`, `exchange`, `bundle`
- `legs_template` -- legs template
- `fixed_terms` / `variable_terms`
- `auto_create_rules` -- auto-creation (by schedule/condition)
- `template_ttl_ms`, `max_instances`

### Circuit breaker (required for auto_create)

- `price_guard` -- pause when price goes outside bounds; oracle data must be no older than `price_guard_max_staleness_ms` (recommended: 60000, i.e. 1 minute).
- `max_consecutive_failures` -- pause after N failed deals (recommended: 3).
- `max_daily_spend` -- daily spend limit.
- `counterparty_health_check` -- only if counterparty active.

### Template status and API

- `template_status`: `active` | `paused` | `expired`. Gateway automatically sets `paused` when circuit_breaker triggers.
- `POST /template/create` -- create template.
- `GET /template/:id` -- get template.
- `POST /template/:id/instantiate` -- create Deal from template with `variable_terms`.
- `POST /template/:id/pause` -- manual pause; `POST /template/:id/resume` -- resume (with circuit_breaker check).
- `DELETE /template/:id` -- delete template.
- `GET /agent/:id/templates` -- list agent templates.

## Scheduled Deals

Intent with delayed activation.

### Schedule structure

- `scheduled_activation_at_ms` -- exact activation time (one-time scheduled deal).
- `recurring_schedule`:
  - `interval_ms`, `start_at_ms`, `end_at_ms` (optional), `max_occurrences`
  - `schedule_mode`: `fixed` | `sliding`. **fixed** -- activations at schedule (`start_at_ms + N * interval_ms`); if Deal finishes late, Deal N+1 at next slot or immediately with `delayed` flag (missed slots do not accumulate). **sliding** -- next activation from previous Deal close (`previous_deal_closed_at_ms + interval_ms`). Default: `fixed`.
- `schedule_budget` -- maximum total budget for all scheduled deals.

### Lifecycle

1. Intent with `schedule` -> status `scheduled` (not `open`).
2. When `scheduled_activation_at_ms` is reached, gateway performs atomic activate-and-check: verifies Intent `conditions`; if met -> `open`, else retry via `scheduled_condition_recheck_interval_ms`, then `cancelled` by `scheduled_condition_timeout_ms`.
3. For recurring: after Deal closure, next activation per `recurring_schedule`; each also goes through activate-and-check.
4. Budget exhausted -> `cancelled`.

## Deal Chaining (lightweight)

`on_close_triggers[]` in Deal terms:

- `trigger_type`: `create_intent`, `create_deal_from_template`, `notify_agent`.
- `trigger_condition`: `success`, `failure`, `any`.
- `trigger_data_mapping` -- mapping of results to new Intent parameters.
- `max_triggers_per_deal`: 5.
- `max_trigger_chain_depth`: 10.

## Deal Pipelines (DAG)

Complex workflows with linked deals.

### Structure

- `pipeline_id`
- `steps[]`:
  - `step_id`
  - `intent_template`
  - `depends_on[]`
  - `condition`
  - `on_failure`: `abort_pipeline`, `skip_step`, `retry_with_alternatives`
- `pipeline_timeout_ms`
- `pipeline_max_budget` (settlement + fees + gas)
- `pipeline_contingency_budget` (recommended: 10%)

### Lifecycle

```
created -> executing -> completed (or failed, partially_completed)
```

On abort: uncompleted deals cancelled, completed are irreversible.

## Delegation

### Structure

- `delegation_id`
- `delegator_agent_id` / `delegate_agent_id`
- `permissions`: `negotiate`, `settle`, `dispute`, `pipeline_manage`, `discovery_publish`
- `constraints`: max amount, asset types, validity period
- `expires_at_ms`, `delegator_signature`

### Rules

- Sub-agent signs with own key, includes `delegation_id`.
- For settlement: delegator wallet.
- `settlement_allowance` -- pre-authorized on-chain.
- `partial_revoke_delegation` -- partial revocation of permissions without revoking the entire delegation.
- No re-delegation.
- `max_active_delegations_per_agent`: 10.

### DelegationAllowance Contract (on-chain)

For TON coin: smart contract where delegator deposits funds. Sub-agent initiates transfer to escrow (up to limit).

| Method | Description |
|---|---|
| `deposit(amount)` | Top up allowance |
| `withdraw(amount, delegator_signature)` | Withdrawal by delegator |
| `delegate_transfer(deal_id, escrow_address, amount, delegate_signature)` | Transfer to escrow |
| `get_allowance()` | Current balance |
| `revoke(delegator_signature)` | Revoke allowance |
| `emergency_freeze(delegator_signature)` | Instant freeze |

- For jettons: standard `approve` + `transferFrom`.
- `emergency_freeze` -- single signature, no waiting for off-chain revoke.
- On `revoke_delegation` off-chain -- gateway automatically initiates `emergency_freeze` on-chain (if pre-signed freeze exists).
- If pre-signed freeze not provided -> event `DelegationRevokedPendingOnchain` (URGENT).

### Delegation and Pipeline

- Sub-agent with `pipeline_manage` permission can create/cancel pipeline.
- `pipeline_max_budget` must be <= `constraints.max_amount`.
- Total spending of all active pipelines is counted toward the overall limit.

### Multi-signature

- `multi_sig_policy` in Agent Card: threshold scheme (2-of-3).
- For: deals above threshold, key rotation, delegation creation.
- `multi_sig_collect_timeout_ms` for signature collection.

## Capability Description Language

### Capability Schema Registry

- Gateway maintains registry of standard schemas.
- Schema: `schema_id`, `schema_version`, `description`, `input_spec`, `output_spec`, `category`.
- Machine-readable: automatic compatibility detection.

### Standard categories

| Category | Description |
|---|---|
| `trade` | Asset exchange |
| `data_feed` | Data provision |
| `compute` | Computational services (AI, processing) |
| `storage` | Data storage |
| `messaging` | Message aggregation/forwarding |
| `bridge` | Cross-chain operations |
| `custom` | Custom schema |

### Input/Output specification

- `input_spec` / `output_spec` -- JSON Schema.
- `quality_metrics` -- metrics (latency, accuracy, throughput).
- `pricing_model`: `per_request`, `per_unit`, `per_time`, `flat`.

### Schema versioning

- Schemas are versioned by semver.
- Minor updates are backward compatible (addition of optional fields).
- Major updates create a new `schema_id`.
- An agent may support multiple versions of the same schema.

## Capability Composition Engine

Automatic workflow construction from capability chain.

### Capability Chain Request (`POST /capability-chain/resolve`)

- `chain_steps[]`: ordered capabilities with input_mapping, quality_requirements, max_price.
- `total_budget`, `urgency`, `auto_execute`.

### Resolution

1. Discovery for each step.
2. Input/output compatibility check (JSON Schema).
3. Return `CapabilityChainProposal` with agents and cost.
4. Confirmation -> create Pipeline.

## Dynamic Capability Negotiation Protocol (DCNP)

Autonomous capability discovery without human involvement.

### Capability Request

- `POST /capability/request`: requirement description, `requested_input_spec` / `requested_output_spec` (JSON Schema), `quality_requirements`, `budget_range`.
- `discovery_mode`: `broadcast` (send to all suitable agents), `targeted` (specific `target_agent_ids[]`), `chain` (integration with Capability Composition Engine).
- `response_timeout_ms` (recommended: 30000).
- `auto_select`: automatic selection of best response by `selection_criteria` (`price` | `quality` | `speed` | `trust` with weights).

### Flow

1. Initiator creates capability request.
2. Gateway distributes request per `discovery_mode`.
3. Agents respond with `POST /capability/respond` (can_fulfill, proposed_price, proposed_sla, proposed_settlement_mode).
4. If `auto_select: true` -- selection by `selection_criteria` and Deal creation; otherwise manual selection via `POST /capability/request/:id/select`.

### State machine

```
request_created -> collecting_responses -> responses_ready -> deal_negotiating -> deal_created
```

Or `no_match`, `cancelled`.

## Service Delivery Profiles

### service_hash_receipt

- Deliverable fixed by content hash.
- Acceptance by hash match and SLA.

### service_oracle_verifiable

- Result verified by oracle metric.
- `oracle_quorum_min`, `oracle_source_diversity`.

### service_milestone

- Deliverable split into milestones.
- Settlement per milestone via milestone_settlement.

### service_streaming

- Stream delivery (chunks).
- Incremental hash chain for integrity.
- Settlement by chunk batches.

### service_subscription

Recurring delivery with configurable billing model.

**Billing models:**

| Model | Description |
|---|---|
| `fixed_period` | Fixed amount per period. Settlement via milestone_settlement |
| `per_use` | Pay per usage. Usage events signed by provider. Batch release by billing_cycle_ms |
| `tiered` | Tiered pricing: `pricing_tiers[]` with ranges and price per unit |
| `usage_capped` | Per-use up to cap. After cap -- no charge until end of period |

- For `per_use`/`tiered`/`usage_capped`: mandatory `usage_report` every billing_cycle_ms.
- Missed delivery = grounds for partial refund.
- `min_subscription_periods` / `max_subscription_periods`.

**Subscription cancellation:**

- Buyer: `POST /deal/cancel` with `cancel_type: subscription_cancel`. Takes effect after current period ends.
- If `min_subscription_periods` not met -- `early_termination_fee`.
- `early_termination_fee_deposit` is locked at creation as `termination_bond`. Returned upon fulfilling min_periods.
- Provider: notice `provider_cancel_notice_periods` in advance (minimum 1 period). No fee.
- Mutual cancel: immediate, prorating current period.

### Execution timing (required timers for service)

| Timer | Description |
|---|---|
| `execution_start_timeout_ms` | Maximum time from accepted to execution start |
| `execution_heartbeat_ms` | Heartbeat interval from provider |
| `heartbeat_miss_tolerance` | Allowed consecutive heartbeat misses (recommended: 3) |
| `sla_seconds` | Total time for complete execution |

On heartbeat miss gateway sends `HeartbeatMissed`. When `miss_count >= heartbeat_miss_tolerance` -- buyer gains the right to dispute `non_delivery`.

### Acceptance window

Minimum `acceptance_window_ms` values by type:

| Asset type | Minimum | Reason |
|---|---|---|
| coin/jetton | 60000 (1 min) | Quick verification |
| nft | 1800000 (30 min) | Metadata, authenticity, royalty check |
| service | 3600000 (1 hour) | Quality check |

- If buyer does not respond within window -- auto-accept.
- If rejected -- must specify `rejection_reason`.
- `max_rejections_per_deal` limits the number of rejections.

Protection against auto-accept while offline:

- When `capacity_status: offline` during acceptance -- timer is paused.
- Paused up to `acceptance_offline_grace_ms` (recommended: 1 hour).
- If not returned -- auto-accept with `auto_accepted_while_offline` flag, extended dispute window (x2).

Protection against coordinated offline attack:

- Offline within 1 minute of receiving deliverable -- `suspicious_offline` flag.
- Repeat pattern (>3 times in 30 days) -- risk flag `acceptance_avoidance_pattern`, trust_score reduction.
- Provider may specify `require_online_acceptance: true`.

### Quality attestation

For `service_hash_receipt` and `service_streaming`:

- `quality_score` (0-100) on acceptance.
- Affects provider `trust_score`.
- Score below `quality_dispute_threshold` automatically opens dispute.

### Service Pricing Benchmark Oracle

Specialized oracle for service price calibration:

- `data_type: service_pricing`.
- Benchmark metrics: `compute_price_per_flop`, `storage_price_per_gb_month`, `inference_price_per_token`, `data_feed_price_per_request`, `bandwidth_price_per_gb`.
- Computed from weighted average price of closed deals over `benchmark_window_ms` (recommended: 24 hours).
- `GET /oracle/benchmarks?category=compute` -- benchmark request.

### Acceptance Rule DSL

Machine-readable acceptance criteria:

```
rule := condition | condition AND rule | condition OR rule | NOT condition
condition := metric_ref comparator value
comparator := == | != | > | >= | < | <=
metric_ref := output.field_path | metric.metric_name | hash(output) | size(output)
```

Examples:
- `metric.latency_ms < 500 AND metric.accuracy >= 0.95`
- `hash(output) == expected_hash`
- `output.format == "json" AND size(output) > 0`

Built-in functions: `hash(x)`, `size(x)`, `count(x)`, `contains(x, substring)`.

## Agent Groups

### Structure

- `group_id`, `group_name`, `members[]`, `admin_agent_id`
- `group_stake` -- shared stake
- `group_capabilities` -- combined capabilities
- `routing_policy`: `round_robin`, `least_loaded`, `primary_backup`, `capability_based`

### Deal model

- Deal party is always a specific member, not group.
- Reroute possible only before Deal creation.
- Delegation between members for backup.

### Stake model

- `member_stake_contribution` on join.
- `max_pool_slashing_per_member`: 20% of group_stake.
- When exceeded -- member excluded.

## Encrypted Negotiation Channel

### Encryption modes

- `none` -- payload in plaintext (default MVP).
- `gateway_transparent` -- TLS in transit.
- `e2e_encrypted` -- end-to-end between agents.

### E2E protocol

1. Ephemeral public key exchange (X25519).
2. Shared secret via Diffie-Hellman.
3. XChaCha20-Poly1305 encryption.
4. Gateway sees only metadata envelope.

E2E applied to: DM, capability requests, mediation proposals, custom messages (`message_type: encrypted_payload`).
NOT applied to: negotiation (quote/accept), settlement, proof.

For `compliance_mode: regulated`, `key_escrow` may apply: session encryption key is deposited with gateway in encrypted form; access only on regulator request with audit trail. In dispute, parties provide decrypted messages as evidence. Recommended: `e2e_key_ratchet` -- refresh session key every N messages (recommended: 100) for forward secrecy.

## Direct Messaging

Messages outside deal context:

- `capability_inquiry`, `availability_check`, `custom_message`.
- Via gateway (for audit and rate limiting).
- `direct_message_rate`: 5 rps per pair.
- `dm_policy` in Agent Card: `contacts_only` or open.

## Protocol Topology

### Single Gateway (MVP)

- One gateway, all operations.
- Settlement and escrow on-chain.

### Federated Gateways

- Multiple independent gateways. Cross-gateway deals via on-chain escrow (neutral, not owned by any gateway).
- **Gateway Card** (counterpart of Agent Card for gateway): `gateway_id`, `gateway_public_key`, `federation_endpoint_url`, `supported_protocol_versions`, `fee_schedule`, `slo_published`, `active_agents_count`, `gateway_signature`. Peers discover each other via on-chain Gateway Registry or bootstrap list.
- Federation handshake: mutual cryptographic challenge-response (no private keys disclosed). After mutual authentication peer is added to federation. Heartbeat: `federation_heartbeat_ms` (recommended: 60000).
- Discovery federation: on `GET /market/discovery` gateway may include results from peers (`source_gateway_id`). Message relay: envelope + `relay_header` (`source_gateway_id`, `destination_gateway_id`, `relay_signature`).
- **Federation API:** `POST /federation/handshake`, `GET /federation/peers`, `POST /federation/relay`, `GET /federation/:gateway_id/discovery`, `POST /federation/heartbeat`; for dispute: `POST /federation/dispute/relay`, `GET /federation/:gateway_id/deal/:deal_id/audit`, `POST /federation/dispute/evidence`.
- Reputation portability: agent requests signed `reputation_proof` from home gateway (agent_id, trust_score, total_deals_completed, total_volume, dispute_rate, score_timestamp_ms, gateway_id, gateway_signature). When interacting with agents on another gateway, proof is presented; receiving gateway verifies signature. `reputation_proof_ttl_ms` (recommended: 24 hours). `foreign_reputation_discount` (recommended: 0.7). On migration: new gateway requests `reputation_export` from old; old provides within `reputation_export_timeout_ms`. `gateway_federation_score` for reputation among peers.

**Federation trust revocation:** any peer may initiate `POST /federation/:gateway_id/blacklist-propose`. Exclusion requires a quorum of votes (`federation_blacklist_quorum`, recommended: majority, min 3). Excluded gateway: cross-gateway deals -> `paused_settlement`, agent migration recommended.

**Cross-gateway dispute:** on-chain escrow is neutral. Dispute is relayed between gateways via federation. Arbitrator is chosen from a pool not affiliated with either gateway; on assignment a `federation_arbitrator_token` is issued for read-only access to audit on both gateways. Decision is executed on-chain via escrow `dispute_release`.

### Migration between topologies

Migration from Single Gateway to Federated does not require changes to Agent Card or Deal format. On-chain contracts (escrow, settlement) are the same for all topologies. Agents are not required to know the current topology; it is a gateway implementation detail.

### Gateway SLA Bond

- On-chain bond guaranteeing SLO.
- Slashing on downtime, state loss, latency.
- `min_gateway_sla_bond`: recommended 10000 TON. When below threshold gateway goes to `degraded` and receives `bond_replenish_deadline_ms` (recommended: 7 days) to replenish; if not met -- `draining`.

### Gateway Recovery

- Sources: (1) on-chain state (source of truth), (2) audit event log, (3) agent-side state for cross-verification.
- `state_checkpoint` -- periodic snapshot of off-chain state (`checkpoint_interval_ms`, recommended: 60 sec). Checkpoint: `checkpoint_id`, `timestamp_ms`, `state_hash`, `active_deals_count`, `active_sessions_count`, gateway signature. Retain at least 72 hours.
- On recovery: event `GatewayRecoveryStarted`, gateway in `maintenance`, all timers paused. After recovery: `GatewayRecoveryCompleted` with `recovered_deals_count`, `state_hash`. Agents verify deals via `GET /deal/:id`. Conflicts: on-chain > signed messages > gateway audit log.
- `max_recovery_time_ms` (recommended: 1 hour). When exceeded -- right to cancel without penalty, federation reputation penalty. In `GET /protocol/health` during recovery: `recovery_eta_ms`.

### Gateway graceful shutdown (deprecation)

- Event `GatewayDeprecationNotice` with `shutdown_deadline_ms` (minimum 30 days). Gateway in `draining`: no new agents or deals. Active deals continue settlement. Agents migrate via `POST /agent/migrate`. Before shutdown gateway provides `reputation_export` to all agents. Audit trail retained read-only for at least 1 year.

### Agent State Commitments

- Agent periodically sends `POST /agent/:id/state-commitment` with `state_hash`, `deals_merkle_root`, `timestamp_ms`, signature. `state_hash = hash(active_deals_state || session_state || reputation_snapshot)`. `state_commitment_interval_ms` (recommended: 5 minutes). On recovery gateway compares state with latest commitments; on mismatch -- request agent local state. Merkle root allows verifying individual deals without disclosing full state. Optional but gives bonus to recovery priority and trust_score.

### Chain Downtime

- Gateway in `degraded`. Off-chain operations continue. Settlement queued in `settlement_queue`. Timers `settlement_timeout_ms`, `funding_timeout_ms`, `escrow_timeout_ms` paused. Events `ChainDowntimeDetected`, on restore `ChainRestored`. Queue processed by priority (see Priority Settlement Queue). When downtime > `max_chain_downtime_ms` (recommended: 1 hour) -- right to cancel without penalty.

## Data Privacy

- Store only necessary data.
- Pseudonymization in public layer.
- Real IDs only in compliance layer.

### Privacy vs Anti-collusion (two-layer model)

- **Public layer:** discovery, public API -- only `pseudonymous_id` (one-way hash of `agent_id`).
- **Compliance layer:** gateway and arbitrators -- real `agent_id` for anti-collusion analysis. Access is audit-logged.
- Risk flags are published without revealing underlying data (`collusion_risk: high`, but without relationship details).
- For `regulated` compliance_mode: disclosure of real data upon regulator request.

### Recommended retention periods

| Category | Period |
|---|---|
| Trading and settlement events | 5 years |
| Dispute materials | 5 years |
| Collusion analysis data | 2 years |
| Technical transport logs | 90 days |

---

Document version: 1.0
Date: 2026-02-14
