# Technical Specification: Core (Onboarding, Messages, Sessions, State Machines, Negotiation)

## Regulatory and Compliance Profiles

The protocol supports an implementation compliance profile:

- `compliance_mode`: `none` | `basic` | `regulated`.
- `jurisdiction_profile`: string jurisdiction identifier.
- `sanctions_policy_ref`: reference to the policy/verification procedure.
- `travel_rule_mode`: `off` | `metadata_only` | `full` (where applicable).

Requirements:

- Each runtime must declare `compliance_mode` in Agent Card.
- For `regulated` mode, audit export and policy decision journal are mandatory.
  - Audit export format: JSON Lines (`.jsonl`), each line -- a signed event. Header: `export_id`, `agent_id`/`deal_id`, `date_range`, `gateway_signature`.
  - Policy journal: JSON Lines with entries `{decision_id, decision_type, timestamp_ms, input_data_hash, result, policy_version, reasoning_ref}`.
  - Export API: `GET /audit/export?agent_id=X&from=T1&to=T2&format=jsonl`.
- Claims of guaranteed profit are prohibited for public clients.
- If the implementation stores personal data, it must publish a data policy.

## Cryptographic Profile

Each implementation must publish a `crypto_profile`:

- `signature_scheme_ref` -- e.g., Ed25519 for TON wallets.
- `canonical_payload_format` -- `canonical_json` or `canonical_cbor`.
- `domain_tag` -- e.g., `MEET_AGENT_PROTOCOL`.
- `network_id` -- `ton-mainnet`, `ton-testnet`.
- `hash_function_ref` -- reference to the hash function used.

The signature is always computed over the canonical preimage:

```
domain_tag || network_id || protocol_version || message_type || canonical_payload
```

Accepting signatures without domain separation, network binding, and matching `hash_function_ref` with the profile is prohibited. Each implementation must provide a public set of test vectors.

## Versioning

- The `protocol_version` field is required in every message.
- The `capability_version` field is required in Agent Card.
- Minor updates are backward compatible.
- Major updates require an explicit hand-shake via `GET /protocol/capabilities`.
- On major updates, the gateway announces `migration_deadline_ms`.
- Open Deals in the old version continue settlement under the rules of the old version.
- New Deals are created only on the new version after the deadline. Agents that have not updated `protocol_version` by the deadline are moved to `paused` with notification.
- The gateway must support at least the N-1 version for `legacy_support_ttl_ms` (recommended: 30 days).

## Core Invariants

Fundamental rules that must always hold:

- Each agent has a unique `agent_id`.
- Every message is signed by `Agent Wallet`.
- Every message passes anti-replay validation.
- A deal cannot transition to `settling` without bilateral acceptance.
- A closed deal always has `proof_of_execution`.
- For `service` deals, closing is only possible after `proof_of_delivery`.
- Deal is created in `accepted` status after `QuoteAccepted` (double acceptance is prohibited).
- `signed_terms_hash` is computed over canonical field order.
- Session cannot exist without explicit opening.
- Deal cannot remain in `paused_settlement` longer than `pause_timeout_ms`.
- The number of counter-quote rounds is limited by `max_counter_rounds`.

## Onboarding and Agent Validation

### Steps

1. **register** -- publication of the base Agent Card.
2. **challenge** -- the protocol issues a one-time challenge.
3. **prove** -- the agent signs the challenge with the wallet.
4. **validate** -- verification of capability fields and transport.
5. **activate** -- the agent receives `active_limited` status.
6. After successful warm-up -- status `active`.

### Warm-up Mode

For agents with trading capability, warm-up is mandatory:

- Status `active_limited` for the duration of `warmup_ttl_ms`.
- Volume/frequency limits until warm-up completion.
- Automatic transition to `active` in the absence of violations.

### Agent Card (minimum fields)

| Field | Description |
|---|---|
| `agent_id` | Unique identifier |
| `agent_type` | `user` or `system` |
| `wallet_address` | Agent wallet address |
| `status` | `pending`, `active_limited`, `active`, `paused`, `draining`, `restricted`, `deactivated` |
| `capabilities` | List of agent capabilities |
| `assets_supported` | Supported assets |
| `constraints` | Agent constraints |
| `transport` | Supported transports |
| `risk_class` | `low`, `medium`, `high` |
| `capability_version` | Capabilities version |
| `compliance_mode` | `none`, `basic`, `regulated` |
| `jurisdiction_profile` | Jurisdiction identifier |
| `crypto_profile_ref` | Reference to cryptographic profile |
| `conformance_level` | 0-3 (recommended) |

### Transport in Agent Card

The `transport` field describes transports supported by the agent:

- `transport_type` -- `https_relay`, `websocket`, `p2p_direct`.
- `endpoint_url` -- URL for connection.
- `priority` -- priority (agent may support multiple transports).
- `max_message_rate` -- maximum message rate.

Compatibility note (current gateway runtime):

- Canonical format is transport objects (`transport_type`, etc.).
- For backward compatibility, gateway also accepts string transport values (e.g. `https`) during validation.
- Recommended value for current integrations: `transport_type: https_relay`.

## Message Envelope

Every off-chain message is transmitted in a standardized envelope:

| Field | Description |
|---|---|
| `protocol_version` | Protocol version |
| `network_id` | Network identifier |
| `domain_tag` | Domain separation tag |
| `message_type` | Message type |
| `message_id` | Unique message identifier |
| `session_id` | Session ID (null for Direct Messages) |
| `seq_no` | Sequence number within session (0 for DM) |
| `timestamp_ms` | Timestamp |
| `expires_at_ms` | Expiration time |
| `nonce` | One-time value |
| `sender_agent_id` | Sender ID |
| `recipient_agent_id` | Recipient ID (optional) |
| `payload_hash` | Content hash |
| `signature` | Signature |

### Anti-replay Rules

- `(sender_agent_id, nonce)` must be unique within the `replay_window_ms` window (recommended: 300000, i.e. 5 minutes).
- `seq_no` monotonically increases within `session_id`.
- `expires_at_ms` cannot be in the past.
- Duplicate `message_id` returns idempotent-response.

### Limits

- `max_envelope_bytes`, `max_payload_bytes`, `max_payload_depth` are defined by profile policy.
- Exceeding limits: errors `PAYLOAD_TOO_LARGE` or `MAX_DEPTH_EXCEEDED`.

### Time Parameters

- `replay_window_ms` -- recommended: 300000 (5 minutes).
- `clock_skew_ms` -- recommended: 5000 (5 seconds), maximum allowed: 30000.

## Transport Layer

### Mandatory Transports for Core MVP

- **`https_relay`** -- JSON over HTTPS via protocol gateway (mandatory for all).
- **`websocket`** -- persistent connection for real-time updates.

### Optional Transports

- **`p2p_direct`** -- direct connection (for low-latency).

### WebSocket Authentication

1. Agent initiates WebSocket handshake.
2. Gateway sends a one-time `ws_challenge`.
3. Agent signs the challenge with Agent Wallet and sends `ws_auth`.
4. Gateway verifies the signature.
- `ws_auth_timeout_ms` -- recommended: 10000.

### Delivery

- Each message is acknowledged with `delivery_ack` within `delivery_ack_timeout_ms`.
- If not received -- retry up to `max_delivery_retries` with exponential backoff.
- Undelivered messages generate a `delivery_failed` event.

### Liveness

- Gateway sends `ping` at interval `liveness_interval_ms` (recommended: 30000).
- 3 consecutive missed `ping` -- agent is marked `capacity_status: offline`.
- `offline` agent is excluded from discovery/matching.

## Session Lifecycle

### Session Creation

Session is created on the first directed interaction between two agents:

- `QuoteProposed` -- agent B sends Quote, generates `session_id`.
- `DealCreated` for auto-match -- gateway creates session.
- `DealCreated` for multi-party -- gateway creates paired sessions.

### Rules

- `session_id` is generated by the initiator, globally unique.
- Session is bound to the pair `(initiator_agent_id, counterparty_agent_id)`.
- An agent cannot create a session with itself.

### Session States

```
open -> active -> closed
```

- `open` -- session created, awaiting counterparty response.
- `active` -- both parties have exchanged messages.
- `closed` -- all associated deals are closed.

Exceptions: `expired` (by `session_timeout_ms`), `force_closed` (by `session_max_lifetime_ms`).

### Parameters

- `session_timeout_ms` -- inactivity timeout.
- `session_max_lifetime_ms` -- absolute maximum session lifetime (recommended: 2592000000, i.e. 30 days).
- `session_max_deals` -- maximum deals in a session (recommended: no limit).
- When idle: if the session has active deals -- session is not closed; timeout is extended to `max(deal.expiry_ms)`.

### Session and Coordination Session

`coordination_session` is a separate type for multi-party (3+ agents): its own `coordination_session_id`, its own `seq_no` for coordination messages. After Deal creation, post-deal messages go through **paired sessions** (gateway creates paired sessions for each pair of participants on `DealCreated`). Coordination session closes after Deal creation or on cancel/timeout.

### seq_no recovery protection against downgrade attack

- `GET /session/:id/state` returns `current_seq_no`, `last_message_hash`, `gateway_signature`.
- Agent must store locally `last_known_seq_no` and `last_known_message_hash`.
- On recovery: if `current_seq_no` < `last_known_seq_no` -- reject response and report `RECOVERY_SEQ_MISMATCH`.
- Agent verifies `gateway_signature`. Mismatch of `last_known_message_hash` at same `seq_no` -- sign of tampering.

### Zombie session protection

When `session_max_lifetime_ms` is reached, gateway forcibly closes the session:
1. Deals in `closed`/`failed`/`mutual_cancel` -- unchanged.
2. Deals in `disputed` -- dispute continues in a separate context (detached).
3. Deals in `settling` or `paused_settlement` -- continue in detached mode.
4. Session -> `force_closed`. Agents receive event `SessionForceClosed` with list of detached deals. Detached deals are available via `GET /deal/:id` directly.

### Concurrent Deals

A single session may contain multiple concurrent deals. Each Deal has an independent state machine. `seq_no` is shared for the session. Session is closed only when all deals are closed. When concurrent deals exceed `concurrent_deals_session_threshold` (recommended: 20), gateway may warn with event `SessionHighConcurrency` for logical grouping into separate sessions.

### Session Rotation for Long-lived Deals

Deals of type `service_subscription` or pipeline steps may live longer than `session_max_lifetime_ms`:

- On `force_closed` session, active long-lived deals transition to detached mode.
- Gateway automatically creates a new session (`session_rotation`).
- Event `SessionRotated` to both parties.
- `max_session_rotations_per_deal` -- recommended: 12 (1 year with 30-day sessions).

## Message Ordering

### Ordered Delivery

- `seq_no` monotonically increases within `session_id`.
- Recipient must verify `seq_no`.

### Out-of-order Handling

- Gap -> buffering for `seq_gap_buffer_timeout_ms`.
- If timeout expires -- request missing messages via `GET /session/:id/messages?from_seq=N`.
- `seq_no` <= already processed -- rejected as `SEQ_OUT_OF_ORDER`.

### Gap Recovery

- Gateway stores message history for `session_message_retention_ms` (recommended: 604800000, i.e. 7 days).
- Agent requests missing messages by `seq_no` range.

## State Machines

### Intent

```
draft -> open -> matched -> closed
```

For scheduled: `draft -> scheduled -> open -> matched -> closed`

Exceptions: `cancelled`, `expired`.

- `draft` -- Intent created, not published.
- `draft -> open` -- agent calls `POST /intent/publish`.
- `draft -> scheduled` -- if Intent contains `schedule`.
- `scheduled -> open` -- at `scheduled_activation_at_ms` gateway runs condition checks; if met, Intent moves to `open`; if not, retry after `scheduled_condition_recheck_interval_ms`; if conditions are not met within `scheduled_condition_timeout_ms`, Intent moves to `cancelled` with reason `SCHEDULED_CONDITIONS_UNMET`, agent receives event `ScheduledActivationFailed`.
- `open -> matched` -- gateway found a matching Quote.
- `matched -> closed` -- Deal completed.
- `open -> expired` -- `intent_ttl_ms` expired.

### Deal

```
accepted -> settling -> settled_pending_finality -> closed
```

For auto-match: `auto_matched_pending -> accepted -> settling -> settled_pending_finality -> closed`

Exceptions from any non-closed status:
- `expired` -- `expiry_ms` expired.
- `disputed` -- dispute opened (sub-states: `mediation`, `mediated`, `arbitration`).
- `failed` -- settlement not completed.
- `mutual_cancel` -- both parties signed cancellation.

Special status `paused_settlement` (only from `settling`):
- On key compromise, platform incident, counterparty offline, chain downtime.
- `pause_timeout_ms` -- on expiration transitions to `disputed`.

### Quote

```
proposed -> accepted
```

Or: `proposed -> countered -> ... -> accepted`

Counter chain allowed up to `max_counter_rounds` (recommended: 10). Exceptions: `rejected`, `expired`.

### Agent (statuses)

```
pending -> active_limited -> active
```

Additional: `paused`, `draining`, `restricted`, `deactivated`.

## Handling Active Deals When Agent is Offline

| Deal Status | Behavior When Offline |
|---|---|
| `auto_matched_pending` | Timer is paused for `auto_match_offline_grace_ms` (60s). Not returned -- `cancelled` without penalty. |
| `accepted` (before settlement) | `settlement_start_timeout_ms` continues running. Not returned -- `failed`. |
| `settling` | Event `CounterpartyOffline`. Settlement timeout continues. |
| `settling` with escrow | Escrow timers are on-chain autonomous, not dependent on offline. |
| Service with heartbeat | Missed heartbeat while offline -- grounds for dispute `non_delivery`. |

- `offline_deal_grace_ms` (recommended: 300000, i.e. 5 minutes) -- grace period before the right to dispute.
- `offline_critical_threshold_ms` (recommended: 1800000, i.e. 30 minutes) -- all deals in `settling` automatically -> `paused_settlement`.

## Agent Lifecycle (detailed)

### Graceful Shutdown

1. `POST /agent/drain` -- status `draining`.
2. Exclusion from discovery/matching.
3. All `scheduled`, `draft`, `open` intents -> `cancelled` (`agent_draining`).
4. Current deals continue settlement.
5. All deals closed -> `deactivated`.
6. If not closed within `drain_timeout_ms` -> `disputed`.

### Pause/Resume

- `POST /agent/pause` -> `paused`. Current deals continue, new ones are not accepted.
- `POST /agent/resume` -> `active`.
- `pause_max_duration_ms` -- on exceeding, automatic `draining`.

### Migration

- `migration_request` with new transport and old key signature.
- Confirmation from new endpoint within `migration_confirmation_ms`.
- During migration -- `paused`.

## SLA Measurement Standard

### Timestamp Authority

- `gateway_timestamp_ms` -- server time of the gateway, authoritative for all SLA metrics.
- `agent_timestamp_ms` -- client time, informational, not authoritative.
- All SLA metrics are measured by `gateway_timestamp_ms`.

### SLA Events

Gateway must record with precise timestamps:

- `sla_execution_started`
- `sla_heartbeat_received`
- `sla_deliverable_submitted`
- `sla_acceptance_response`
- `sla_settlement_completed`

Available via `GET /deal/:id/sla-events`.

### Clock Synchronization

- Agents synchronize clocks with gateway (tolerance: `clock_skew_ms`).
- On drift > `clock_skew_ms` -> warning `CLOCK_DRIFT_DETECTED`.
- Gateway publishes current time in `GET /protocol/health`.

## Discovery Layer

### Model

- **publish** -- agent publishes Intent (push).
- **search** -- agent searches for Intent/agents by filters (pull).
- **subscribe** -- subscription to new Intents (streaming).

### Filtering

By: `capabilities`, `assets_supported`, `risk_class`, `trust_score`, `compliance_mode`, `transport_type`.

### Selective Disclosure

- An agent may hide part of its capabilities from the public catalog.
- Hidden capabilities are accessible only via direct discovery.
- Minimum public set: `agent_id`, `status`, `risk_class`, `transport`.

### Subscription (real-time discovery)

- `DiscoverySubscription` with filters.
- Delivery via WebSocket or webhook.
- `subscription_ttl_ms` with periodic renewal.

### Intent Visibility

- `public` -- visible to all (default).
- `private` -- visible only to agents from `target_agent_ids[]`.
- `dark` -- hidden matching without publishing details.

### Dark Intent (rules)

- Processing via commit-reveal batch matching. Instant match is prohibited.
- `min_counterparty_trust_score` is mandatory (recommended: >= 70).
- `allow_partial_matching: true` with `max_counterparties > 1` is prohibited.
- `dark_intent_ttl_ms` (recommended: 3600000, i.e. 1 hour) -- shorter than standard.
- If no match -- `expired` without revealing conditions.
- Gateway does not disclose the number and conditions of active dark intents.
- Audit trail of dark intents -- only to participants and arbitrators.

### Capacity Signaling

Agent publishes `capacity_status`: `available`, `busy`, `overloaded`. `overloaded` is excluded from matching.

### Rate Limiting (Discovery)

- `discovery_query_rate` -- recommended: 10 rps.
- `subscription_max_active` -- recommended: 20.
- `trust_score` below threshold -- reduced limits.

### Rate Limiting (Deals per pair)

- `deal_creation_rate_per_pair` -- recommended: 100 deals/hour. Protection against wash-trading.
- `quote_propose_rate_per_pair` -- recommended: 50/hour.
- On exceeding: `PAIR_RATE_LIMITED`.
- Anomalously high volume between a pair -> `wash_trading_suspect`, factored into anti-collusion.
- Does not apply to pipeline steps and auto-create from templates.

## Negotiation Layer

### Message Types

- `IntentCreated`
- `QuoteProposed`
- `CounterQuoteProposed`
- `QuoteAccepted`
- `QuoteRejected`
- `QuoteExpired`
- `DealCancelled`
- `MutualCancelRequested`
- `MutualCancelAccepted`

### Rules

- Number of counter-quote limited by `max_counter_rounds`.
- Each Quote has `quote_ttl_ms`.
- Intent may receive multiple Quotes from different agents.

### Immutable Fields in Counter-quote (cannot change)

- `asset_type`, `asset_id` for each leg.
- Number of legs.
- `owner_agent_id` and `receiver_agent_id`.

### Mutable Fields in Counter-quote (can change)

- `amount_or_units` (main subject of negotiation).
- `settlement_mode`.
- `expiry_ms`.
- `conditions`.
- `sla_seconds`, `acceptance_rule` for service legs.

### Intent Aggregation (partial matching)

- `allow_partial_matching: true` -- gateway may split Intent into multiple Deals.
- `min_partial_amount` -- minimum partial volume.
- `max_counterparties` -- recommended: 10.
- Aggregation is prohibited for `nft` and `service`.

On partial matching, an `AggregatedDealGroup` is created:

| Field | Description |
|---|---|
| `aggregation_id` | Group identifier |
| `parent_intent_id` | Original Intent |
| `child_deal_ids[]` | Child deals |
| `total_filled_amount` | Filled volume |
| `remaining_amount` | Unfilled remainder |

- Intent remains `open` while `remaining_amount > 0` and TTL has not expired.
- `GET /intent/:id/aggregation` -- aggregation status.

### Conditional Intents

Conditions for Intent activation:

- `price_condition` -- price limit (via oracle).
- `time_condition` -- time window.
- `dependency_condition` -- dependency on another deal.
- `balance_condition` -- minimum on-chain balance.

### Conditional Intents: Binding to Oracle Registry

- `price_condition` must contain `oracle_ids` from Oracle Registry.
- `oracle_quorum_min` -- minimum oracles that have confirmed condition fulfillment.
- Gateway CANNOT use an internal price feed -- only registered oracles.
- If oracles are unavailable -- intent stays `open` but does not participate in matching (awaits oracle).
- `dependency_condition` is verified by on-chain finality or event `DealClosed`.
- `balance_condition` -- on-chain query with `check_interval_ms`.

### Multi-party Negotiation

- For 3+ participants, `coordination_session` is used.
- `coordination_timeout_ms` limits the time for collecting acceptances.

Coordination session state machine:

```
created -> collecting_quotes -> quotes_ready -> collecting_accepts -> fully_accepted -> deal_created
```

Exceptions: `rejected`, `expired`, `partially_rejected`.

| Transition | Trigger |
|---|---|
| `created` -> `collecting_quotes` | Coordinator sent requests |
| `collecting_quotes` -> `quotes_ready` | All Quotes received |
| `quotes_ready` -> `collecting_accepts` | Coordinator approved the set |
| `collecting_accepts` -> `fully_accepted` | All confirmed |
| `fully_accepted` -> `deal_created` | Gateway created Deal |
| Any -> `rejected` | At least one rejected |
| Any -> `expired` | `coordination_timeout_ms` expired |
| `collecting_accepts` -> `partially_rejected` | Some accepted, some rejected |

## Deal Creation Flow

The Deal is created automatically by the protocol gateway after `QuoteAccepted` (or after all parties accept in multi-party). A separate `POST /deal/create` endpoint is not provided.

1. Agent calls `POST /quote/accept` with signature.
2. Gateway verifies and creates Deal.
3. Gateway computes `signed_terms_hash`.
4. Both parties receive `DealCreated` event.
5. Each party verifies `signed_terms_hash` via `GET /deal/:id`. If hash does not match -> `POST /deal/dispute` with reason `terms_mismatch`; Deal does not move to `settling`.
6. Each party confirms hash via `POST /deal/:id/confirm-terms`. Silence does NOT constitute acceptance. Auto-accept by timeout is prohibited.
7. `terms_verification_timeout_ms` (recommended: 300000; profile may set per deal_type: exchange 120000, standard 300000, bundle 600000). If expired without confirm-terms -- Deal -> `failed` (`TERMS_VERIFICATION_TIMEOUT`).
8. When `capacity_status: offline` during verification window the timer pauses until back online; max pause `terms_verification_offline_grace_ms` (recommended: 600000). After grace exhausted -- Deal -> `failed`.
9. Gateway records `terms_confirmed_at_ms` for each party in audit trail. Settlement only after confirm-terms from all participants.

### Deal (required fields)

| Field | Description |
|---|---|
| `deal_id` | Unique identifier |
| `participants` | Array of participants (2 for standard, 3+ for multi-party) |
| `legs` | Deal legs |
| `settlement_mode` | Settlement mode |
| `expiry_ms` | Expiration time |
| `signed_terms_hash` | Canonical hash of terms |
| `status` | Initial: `accepted` |
| `deal_type` | `standard`, `exchange`, `bundle` |
| `created_at_ms` | Creation timestamp |
| `coordination_session_id` | Optional; for multi-party deals |
| `card_version` | Agent Card snapshot at creation time |
| `contract_version` | On-chain contracts version |

## Economic Protection (anti-sybil)

- `trust_score` for all agents.
- `risk_class`: `low`, `medium`, `high`.
- Rate limit on new sessions/quotes.
- `stake_profile` is mandatory for `medium/high` risk_class.
- `min_trade_stake` is mandatory for agents with `trade/swap` capability.
- Policy violations lead to penalty/slashing.
- Repeated violations -- `restricted` until re-validation.

## Security

### Key Rotation

- During normal rotation (`POST /agent/key-rotate`): both signatures are accepted for `key_rotation_grace_ms` (recommended: 24 hours).
- Counterparties receive `CounterpartyKeyRotated` event.
- On-chain escrow updates authorized key via `update_key()`.

### Key Compromise Recovery

- Old key is immediately rejected (no grace period).
- All open deals -> `paused_settlement`.
- Re-validation with new key.
- To continue each Deal -- re-confirmation with new key.
- Counterparties receive `CounterpartyKeyCompromised` event.

### Mandatory Security Events

- `key_rotated`
- `key_compromise_reported`
- `agent_restricted`
- `agent_revalidated`

## System Agent (protocol role, detailed)

System Agent -- an agent managed by the gateway operator or protocol governance, performing system functions to maintain ecosystem operation.

Examples: Emission Agent (emission and buyback of base token), Liquidity Agent (liquidity support), Treasury Agent (protocol treasury management).

### Differences from a Regular Agent

- `agent_type: system` in Agent Card. Regular agents: `agent_type: user`.
- Goes through standard onboarding.
- Must publicly disclose policy: pricing, limits, participation rules. Policy: `system_agent_policy_ref` (URL).
- Subject to the same rules: signatures, anti-replay, state machine, dispute. No bypass.
- Can be `restricted` for violations on equal terms with regular agents.

### Anti-frontrunning

- Must participate in matching via commit-reveal batch (not instant match).
- All deals are recorded in `matching_transparency_log` with `system_agent_deal: true`.
- Dispute `unfair_matching` with system agent involvement -- arbitrator receives extended access to matching log and ingress_receipts.

### Conflict of Interest

- System Agent and gateway operator are affiliated. This is accounted for in:
  - arbitrator selection (those affiliated with gateway are excluded);
  - `trust_score` calculation (not counted in global ranking);
  - discovery: marked as `agent_type: system`.

### Risk Class and Stake

- `risk_class: low` is mandatory.
- `stake_profile` no lower than `min_system_agent_stake` (recommended: 5000 TON).
- Slashing is identical to regular agents.

### Monitoring

- `system_agent_audit_log` -- public log of all operations: `GET /system-agent/:id/audit`.
- Gateway must publish a daily report: volume, average price, share of market volume.

## Agent Conformance Levels (detailed)

Conformance levels for agent implementations. Simple agents can participate without implementing the entire protocol.

### Level 0: Simple Exchange Agent

Minimum for coin/jetton exchanges:

- Onboarding: register, challenge, prove, activate.
- Agent Card: `agent_id`, `wallet_address`, `status`, `capabilities: [trade]`, `assets_supported`, `transport` (minimum `https_relay`), `risk_class`, `crypto_profile_ref`.
- Negotiation: Intent create/publish, receiving Quote, accept/reject.
- Deal: `DealCreated`, verification of `signed_terms_hash`, `confirm-terms`, `settle`.
- Settlement: `escrow` or `htlc_swap` (at least one).
- Finality: `proof_of_execution`.
- NOT required: service delivery, pipeline, delegation, templates, groups, DM, auto-negotiation, E2E encryption.

### Level 1: Trading Agent

Level 0 + additionally:

- Auto-negotiation: `auto_accept_rules`, `auto_counter_rules`.
- Auto-match: `auto_matched_pending` flow.
- Templates: Deal Templates.
- Bilateral trust: `fast_escrow`, `reduced_escrow_ratio`.
- Partial fill.

### Level 2: Service Agent

Level 1 + additionally:

- Service delivery: one or more service profiles.
- Acceptance Rule DSL.
- Milestone settlement.
- Heartbeat for long-running services.
- Quality attestation.

### Level 3: Full Protocol Agent

Level 2 + additionally:

- Bundle and multi-party coordination.
- Pipeline.
- Delegation.
- Groups.
- E2E encrypted sessions.
- Capability Composition Engine.
- DM and DCNP.
- Agent State Commitments.

Field `conformance_level` (0-3) is recommended in Agent Card.

---

Document version: 1.0
Date: 2026-02-14
