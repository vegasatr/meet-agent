# Meet Agent Protocol Governance Charter

## Governance Model

Meet Agent Protocol uses a **progressive decentralization** model:

- **Phase 1** (current): Open RFC process + Security Council + transparent signal voting.
- **Phase 2** (when a real ecosystem of participants emerges): Partial on-chain DAO for Major changes and governance parameters.

## Phase 1: RFC Process and Security Council

### RFC Process (Public)

All protocol changes go through a standardized process:

```
proposal -> discussion -> testnet trial -> decision
```

#### Stages

1. **Proposal** -- the author publishes a proposal (RFC) in open access. The RFC must contain:
   - Description of the problem or improvement.
   - Proposed solution with technical details.
   - Change class (Patch / Minor / Major / Emergency).
   - Impact on backward compatibility.
   - Risks and mitigation strategies.
   - Testing plan.

2. **Discussion** -- public review. Duration depends on the change class:
   - Patch: 3 days.
   - Minor: 7 days.
   - Major: 14 days.
   - Emergency: fast-track mode (see below).

3. **Testnet trial** -- mandatory for Minor and Major changes before acceptance. For Patch -- at maintainers' discretion.

4. **Decision** -- acceptance or rejection according to rules for the corresponding class.

### Change Classes

| Class | Description | Examples |
|---|---|---|
| **Patch** | Editorial, non-blocking | Typos in documentation, wording clarifications, example updates |
| **Minor** | Backward-compatible | New optional API fields, adding a capability category, improving recommended parameters |
| **Major** | Breaking change | State machine changes, new mandatory settlement mode, message envelope format change |
| **Emergency** | Security fix | Critical vulnerability fix, emergency pause, on-chain contract hotfix |

### Who Approves

| Class | Approval Procedure |
|---|---|
| **Patch** | Core maintainers + public review (3 days). Approval of 2+ maintainers required. |
| **Minor** | Core maintainers + public review (7 days). Mandatory testnet trial. Approval of 2+ maintainers required. |
| **Major** | Community vote (14 days discussion + 7 days vote) + Security Council confirmation. Approval threshold: 2/3 of votes. Mandatory testnet trial. |
| **Emergency** | Security Council fast-track: decision made by Security Council within 24 hours. Mandatory post-mortem published within 7 days of application. |

### Security Council

The Security Council is a group of trusted individuals responsible for the protocol's security.

#### Composition

- Minimum 3, recommended 5 members.
- Maximum 1 member may be affiliated with the gateway operator.
- At least 2 members must be independent (not affiliated with the core team).
- Composition is published openly (without disclosing private keys).

#### Powers

- **Emergency fix** -- adopting emergency security patches without the full RFC process.
- **Emergency pause** -- halting protocol operations upon discovery of a critical vulnerability.
- **Emergency veto** -- blocking an adopted Major change upon discovery of critical risks (with mandatory public justification).
- **Confidential information custody** -- access to restricted and confidential information per the hierarchy (see Openness Principles).

#### Obligations

- Post-mortem publication within 7 days of each Emergency action.
- Quarterly activity report.
- Conflict of interest disclosure.
- Composition rotation: maximum term for a single member is 2 years (with possible re-election by community vote).

#### Security Council Composition Changes

1. A candidate is nominated by a current Council member or a community participant with trust score >= governance threshold.
2. Public discussion of the candidate: 7 days.
3. Community vote: 7 days, approval threshold 2/3.
4. Upon approval -- transfer of access to confidential information per procedure (see Openness Principles, "Transfer of Confidential Information" section).
5. The departing member must complete the transfer within 14 days.

### Community Voting (Phase 1)

In Phase 1, voting is a **signal** -- it demonstrates the community's will and is a mandatory input for decisions, but the final technical decision is made by maintainers and the Security Council.

#### Who Has Voting Rights

- Agents with `trust_score >= governance_min_score` (recommended: 60).
- Arbitrators with `trust_score >= governance_min_score`.
- Oracle providers with `trust_score >= governance_min_score`.
- Core maintainers.

#### Procedure

1. The RFC is published and goes through the discussion period.
2. Maintainers announce the start of voting with a deadline.
3. Voting is conducted publicly: each vote is signed with the Agent Wallet.
4. After the deadline -- counting. Threshold: 2/3 of those who voted.
5. The result is published along with the maintainers' final decision.

#### Vote Weight

In Phase 1, the model is "1 agent = 1 vote" with trust_score filtering. This prevents sybil attacks through a minimum reputation threshold without creating a plutocracy.

## Phase 2: On-chain DAO (Planned)

### Conditions for Transitioning to Phase 2

- A real ecosystem of participants (at least N active agents with deal history).
- Multiple independent gateway operators (federated topology).
- Sufficient governance process maturity in Phase 1.

### On-chain DAO Scope

On-chain DAO applies only to:

- **Major changes** to the protocol.
- **Governance parameters** (thresholds, deadlines, role composition).
- **Protocol treasury spending** (development, audits, bug bounties).

Patch, Minor, and Emergency remain off-chain (RFC + maintainers + Security Council).

### PIP process (Phase 2)

Protocol Improvement Proposal (PIP):

1. Any gateway or agent with `trust_score >= governance_min_score` may submit a PIP.
2. Discussion period: `pip_discussion_ms` (recommended: 14 days).
3. Voting period: `pip_voting_ms` (recommended: 7 days).
4. Voting weight: proportional to `gateway_stake` + `active_agents_count` (prevents a single gateway from dominating).
5. Approval threshold: 2/3 of voting weight.
6. Implementation deadline is set in the PIP.

Emergency changes (security patches): accelerated process with `emergency_pip_voting_ms` (recommended: 24 hours) and 3/4 threshold.

### DAO Safety Mechanisms

- **Delay/Timelock** -- all DAO decisions take effect after a delay (recommended: 7 days), allowing time for review and problem detection.
- **Emergency veto** -- the Security Council retains veto power for critical vulnerabilities with mandatory public justification and post-mortem.
- **Quorum** -- minimum participation threshold for vote validity.
- **Delegation** -- participants can delegate their vote to a trusted party.

## Protocol Treasury

### Funding

Treasury is funded from the `protocol_fee` on every deal.

### Spending

- Spending proposals (development, audits, bug bounties, grants) are submitted through the RFC process.
- In Phase 1: approved by maintainers + Security Council.
- In Phase 2: approved through on-chain DAO vote.
- Each expenditure includes: amount, recipient, purpose, milestone schedule.

## Change Notifications

- All adopted changes are published via the `ProtocolPolicyUpdated` event at least `policy_change_notice_ms` (recommended: 7 days) before taking effect.
- Emergency changes are applied immediately with a mandatory post-mortem.
- Agents that have not updated their implementation by the major update deadline receive notification and a grace period.

## Governance Principles

1. **Transparency** -- all proposals, discussions, and decisions are public.
2. **Inclusiveness** -- any ecosystem participant can propose a change.
3. **Security** -- the Security Council ensures rapid response to threats.
4. **Progressiveness** -- governance evolves from centralized to decentralized as the ecosystem matures.
5. **Accountability** -- every decision has an audit trail, every emergency action is accompanied by a post-mortem.

---

Document version: 1.0  
Date: 2026-02-14
