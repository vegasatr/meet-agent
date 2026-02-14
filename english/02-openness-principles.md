# Openness Principles of Meet Agent Protocol

## Philosophy

Meet Agent Protocol is built on the conviction that trust in an autonomous deal protocol is only possible through transparency. Agents managing real funds must operate under rules that anyone can verify.

## What Is Open

### Protocol Specification

The full Core MVP specification is published openly:

- All state machines (Agent, Intent, Quote, Deal, Escrow, HTLC).
- Rules for negotiation, settlement, dispute, finality.
- Canonical terms hash -- format and field order.
- Message envelope and anti-replay rules.
- Acceptance Rule DSL for machine-readable acceptance criteria.
- Settlement Layer: escrow, HTLC swap, milestone, bridge swap.
- API Contract -- the complete set of endpoints.

### Governance

- All protocol changes go through a public RFC process.
- Votes on Major changes are conducted openly.
- Security Council decisions are published with mandatory post-mortems.
- Protocol Improvement Proposals (PIPs) are available for anyone to read and discuss.

### Protocol Economics

- Trust score formula and update rules.
- System Agent (Emission Agent) pricing policy.
- Emission formula and buyback rules.
- Fee policy: protocol fee, gateway fee.
- Slashing rules for all participants.

### Audit and Monitoring

- System Agent deal logs (public audit log).
- Matching transparency log.
- Protocol metrics (Prometheus-compatible format).
- Daily reports on system agent operations.
- Proof publication for every deal.

### Code

- On-chain contracts (Escrow, HTLC, Bundle Coordinator, DelegationAllowance) are published open source after audit.
- Reference implementation SDK is published as it becomes ready.
- Test vectors for the cryptographic profile.

## What Is Not Published

Secret information, keys, and any other information that could help destabilize the protocol or compromise participants is not published.

A certain category of information is not published openly, as its disclosure could be exploited by malicious actors:

- Implementation details of specific anti-abuse and anti-frontrunning mechanisms at the gateway runtime level (detection parameters, trigger thresholds, internal monitoring algorithms).
- Private keys, seed phrases, confidential infrastructure configurations.
- Specific hot-fix strategies for discovered 0-day vulnerabilities before the vulnerability is patched.
- Internal Security Council operational procedures during an active incident.

This information is classified as **confidential** and is managed according to the rules described below.

## Transfer of Confidential Information

### Principle

Confidential information is held by a limited circle of persons (Security Council, core maintainers). When transfer to another person is necessary, strict rules apply.

### Transfer Rules

1. **Initiation** -- transfer of confidential information to a new person is only possible by decision of the Security Council or by community vote (for cases of Security Council composition changes).

2. **Community vote** -- if the transfer is related to a change in the composition of key custodians (e.g., a Security Council member departing), a vote is held:
   - The new custodian candidate is nominated and discussed publicly.
   - Voting is conducted among participants with voting rights (as defined in the Governance Charter).
   - Approval threshold: simple majority for Patch/Minor information, 2/3 for Major information.

3. **Transfer procedure**:
   - Transfer is conducted through a secure channel (end-to-end encrypted).
   - The fact of transfer is recorded in the Security Council's closed audit trail specifying: to whom, when, and what category of information.
   - The receiving party confirms receipt with a signature.
   - The departing custodian is required to delete their copies within an agreed timeframe (if leaving the role).

4. **Access revocation** -- when a person leaves the custodian role:
   - All trusted keys and credentials are rotated.
   - The rotation fact is recorded in the audit trail.
   - Security Council conducts a review of affected systems.

### Access Hierarchy

| Level | Who Has Access | What It Includes |
|---|---|---|
| Public | Everyone | Specification, governance, economics, API, open source code |
| Restricted | Core Maintainers + Security Council | Anti-abuse details, monitoring parameters, operational procedures |
| Confidential | Security Council | Infrastructure private keys, 0-day hotfix details, emergency procedures |

### Access Audit

- The Security Council maintains an access log for confidential information.
- Quarterly review of the roster of persons with access.
- Upon discovery of unauthorized access -- immediate rotation and post-mortem.

## Commitments to the Community

1. **No hidden privileges** -- System Agents follow the same protocol rules as regular agents. Their policy is public.
2. **No hidden emission** -- MEET token parameters are fixed on-chain, ownership revoked.
3. **Public proofs** -- every deal leaves a verifiable proof.
4. **Open arbitration** -- arbitrator decisions include reasoning and references to evidence.
5. **Progressive decentralization** -- moving from a single operator to federated governance and DAO.

---

Document version: 1.0  
Date: 2026-02-14
