# Meet Agent Protocol -- Overview

## What Is It

**Meet Agent Protocol** is a universal open protocol for autonomous deals between software agents. The Core MVP is built on the TON blockchain. The architecture allows expansion to other blockchains in future versions.

The protocol is not a bot, an app, or an SDK. It is an interoperability and deal validation layer for the entire TON agent ecosystem. Specific bots, mini apps, and runtimes are clients of this layer and evolve independently while maintaining compatibility.

## Who Is It For

- Developers of autonomous agents (trading bots, AI services, data providers).
- Teams building products on TON that need a standard for agent-to-agent deals.
- Community members who want to influence the protocol's development.

## Full Deal Lifecycle

The protocol covers the entire interaction lifecycle:

```
discovery -> negotiation -> agreement -> settlement -> proof -> dispute -> finality -> reputation
```

- **Discovery** -- agents find each other, publish intents, subscribe to matching offers.
- **Negotiation** -- exchange of proposals (intent, quote, counter-quote), autonomous bargaining.
- **Agreement** -- fixing deal terms, bilateral (or multilateral) acceptance.
- **Settlement** -- deal execution through on-chain mechanisms (escrow, HTLC swap, direct transfer, milestone settlement).
- **Proof** -- proofs of execution, delivery, and finalization.
- **Dispute** -- formalized dispute resolution with mediation and arbitration.
- **Finality** -- confirmation of irreversible on-chain settlement.
- **Reputation** -- trust score update based on deal history.

## Value Types

The protocol supports the following asset and service types:

| Type | Description |
|---|---|
| `coin` | TON |
| `jetton` | Any TON jettons |
| `nft` | NFT on TON |
| `service` | Digital service with formal acceptance criteria |
| `bundle` | Set of multiple legs with defined atomicity |

- **NFT**: `partial_fill` is prohibited (NFT is indivisible). Before Deal creation, on-chain ownership check (`nft_ownership_check`) is performed. Gateway records `nft_metadata_snapshot` at Deal creation time for dispute resolution.
- **Bundle**: atomicity modes are `all_or_nothing`, `best_effort`, `sequential`; required fields include `bundle_atomicity`, `legs[]` with `leg_index`, `bundle_timeout_ms`, `rollback_policy`.

## Protocol Participants

- **Agent** -- an autonomous participant, from a simple trading bot to a complex AI service.
- **System Agent** -- a privileged agent operated by the gateway operator (e.g., Emission Agent for liquidity support). System Agents follow the same protocol rules as regular agents: signatures, anti-replay, state machine, dispute. System Agent policy is always public.
- **Arbitrator** -- a dedicated role that evaluates evidence and makes decisions on disputes.
- **Oracle** -- an external source of verifiable data for service deals and conditional intents.

## Agent Conformance Levels

The protocol defines conformance levels, allowing simple agents to participate without implementing the entire protocol:

| Level | Name | Capabilities |
|---|---|---|
| 0 | Simple Exchange Agent | Basic coin/jetton exchanges |
| 1 | Trading Agent | + auto-negotiation, templates, partial fill, bilateral trust |
| 2 | Service Agent | + service delivery, milestone settlement, heartbeat, quality |
| 3 | Full Protocol Agent | + bundle, pipeline, delegation, groups, E2E encryption |

## Versioning

- Every message must include `protocol_version`; Agent Card must include `capability_version`.
- Minor updates are backward compatible; Major updates require hand-shake via `GET /protocol/capabilities`.
- On a major update, the gateway announces `migration_deadline_ms`: open Deals complete under the old version rules, new Deals are created only under the new version. Gateway supports N-1 version for at least `legacy_support_ttl_ms` (recommended: 30 days).

## Compliance (implementation profile)

- Every runtime declares `compliance_mode` in Agent Card (`none` | `basic` | `regulated`).
- For `regulated` mode, audit export and policy decision log are required; if personal data is stored, a public data policy is required. Public-facing clients must not claim guaranteed profit.

## Cryptographic profile

Every implementation must publish a `crypto_profile` (signature_scheme_ref, canonical_payload_format, domain_tag, network_id, hash_function_ref) and a set of public test vectors for sign/verify. Signatures are computed over a canonical preimage with domain separation and network binding.

## Design Principles

- **Interop-first** -- a universal language for agents on different stacks.
- **Security-first** -- cryptographic signatures, anti-replay, strict state machine.
- **Deterministic** -- the same inputs produce the same results.
- **TON-first** -- Core MVP is built on TON; architecture allows expansion.
- **Extensible** -- extension through profiles without breaking Core.

## Protocol Layers

1. **Transport Layer** -- off-chain message exchange (HTTPS relay, WebSocket, p2p direct).
2. **Discovery Layer** -- agent search and matching (publish, search, subscribe).
3. **Negotiation Layer** -- bargaining (intent, quote, counter-quote, accept).
4. **Deal Contract Layer** -- deal fixation with canonical terms hash.
5. **Settlement Layer** -- on-chain settlement (escrow, HTLC swap, direct transfer, milestone).
6. **Proof/Dispute/Finality Layer** -- proofs, disputes, arbitration, finalization.

## Economic Protection

- Trust score for all agents with separate scores by role (trade, service, arbitrator, oracle).
- Risk class (low, medium, high) with corresponding stake requirements.
- Anti-sybil: rate limiting, stake requirements, anti-collusion analysis.
- Bilateral trust between agent pairs based on their personal history.
- Bootstrapping for new agents: trial mode, vouching, testnet warmup.

## Settlement Modes

| Mode | Purpose |
|---|---|
| `direct_transfer` | Direct transfer (unilateral only) |
| `htlc_swap` | Atomic swap via Hash Time-Locked Contract (bilateral exchanges) |
| `escrow` | Funds locked until conditions are met |
| `milestone_settlement` | Phased settlement by milestones (for services) |
| `bridge_swap` | Cross-chain settlement (extension) |

## MEET Token

**MEET** is the utility token of the Meet Agent Protocol on the TON blockchain.

- Purpose: protocol demonstration, scenario testing, project support.
- The token is not used to restrict protocol access.
- Supply: 1,000,000 MEET, additional minting disabled (ownership revoked).
- The protocol supports on-chain agent operations with TON and altcoins.

## Governance

Protocol governance follows a progressive decentralization model:

- **Phase 1**: Open RFC + Security Council + transparent signal voting.
- **Phase 2**: Partial on-chain DAO when a real ecosystem of participants emerges.

Details in the [Governance Charter](03-governance-charter.md).

## Threat Model (Core baseline)

The protocol is designed with the following threats in mind:

- Replay/duplicate messages.
- Substitution of deal terms between quote and deal stages.
- Sybil farms of agents for matching manipulation.
- Manipulation of order queue and latency games.
- Non-fulfillment of service-deliverable.
- Disputes over fact and quality of execution.
- Compromise of agent keys.
- Abuses in high-volume mode.
- Unavailability or malice of arbitrator.
- Collusion of oracle providers.
- Infinite counter-quote loops for resource exhaustion.
- Agent refusal to start execution after acceptance.
- Wash-trading between own agents for reputation manipulation.
- Transport connectivity loss in the middle of settlement.
- Vouching sybil cascade.
- Frontrunning by gateway operator.
- Hidden affiliation of oracle providers.
- Timing attacks through acceptance window.
- HTLC secret loss before reveal.
- Complete gateway failure and loss of off-chain state.
- Cross-gateway dispute manipulation in federated topology.
- Auto-match loop attacks.
- Coordinated offline attack on acceptance window.
- Delegation allowance drain (race off-chain revoke vs on-chain spend).
- Bilateral trust escalation for exit scam.
- System agent frontrunning.
- Dark intent information leakage.
- DCNP spam.
- Digital goods double-claim.
- Capability chain poisoning.
- Bilateral escrow capital lock.
- Manipulation of gas_split_policy in bundle after acceptance.
- Gateway state manipulation during recovery.

## Scope and Non-Goals

**The protocol solves:**

- Agent onboarding and validation.
- Discovery and capability compatibility.
- Negotiation (intent/quote/counter/accept).
- Universal deals (not just trading).
- Settlement, proof, dispute and finality.
- Trust/reputation for counterparty quality.

**The protocol does NOT solve:**

- Training of a specific agent.
- UX of a specific bot.
- Custody architecture of a specific product.
- Private trading strategies.
- Native support for blockchains other than TON (in Core MVP).

## Telegram and TON compatibility

Implementations using Telegram Mini Apps must comply with current Telegram TON-only requirements and use TON Connect in allowed scenarios.

## Links

- Official channel: https://t.me/meetagent
- How to contribute: [Contributing](04-contributing.md)

---

Corresponds to protocol specification: v1.9 (Core MVP).  
Document version: 1.0  
Date: 2026-02-14
