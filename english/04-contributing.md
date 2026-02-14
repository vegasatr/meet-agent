# How to Contribute to Meet Agent Protocol

## Who Can Propose Changes

Anyone in the ecosystem can propose a change:

- Agent or bot developer.
- Gateway operator.
- Arbitrator or oracle provider.
- Community participant.

## Contribution Process

### 1. Determine the Change Class

| Class | What It Includes | Review Timeline |
|---|---|---|
| **Patch** | Typos, documentation clarifications, non-blocking edits | 3 days |
| **Minor** | New optional fields, backward-compatible improvements | 7 days + testnet |
| **Major** | Breaking changes, new mandatory mechanisms | 14 days discussion + 7 days vote + testnet |
| **Emergency** | Security fix | Fast-track via Security Council (24 hours) |

### 2. Prepare an RFC (Request for Comments)

The RFC is the primary proposal format. It should contain:

**Required sections:**

- **Title** -- brief description of the proposal.
- **Change class** -- Patch / Minor / Major / Emergency.
- **Motivation** -- what problem it solves, why it is needed.
- **Description** -- technical description of the proposal.
- **Compatibility impact** -- whether it breaks backward compatibility, which agents are affected.
- **Risks** -- potential issues and mitigation strategies.

**Recommended sections (for Minor and Major):**

- **Alternatives** -- what other options were considered and why this one was chosen.
- **Testing plan** -- how the proposal will be verified in testnet.
- **Migration plan** -- how existing agents will transition to the new version.
- **Examples** -- concrete usage examples.

### 3. Publish the RFC

- Post the RFC in the community channel or the appropriate discussion section.
- Tag the change class for proper routing.

### 4. Participate in Discussion

- Respond to questions and comments.
- Revise the RFC based on feedback.
- Be prepared for a testnet trial for Minor and Major proposals.

### 5. Await the Decision

| Class | Who Decides |
|---|---|
| **Patch** | 2+ core maintainers |
| **Minor** | 2+ core maintainers after testnet trial |
| **Major** | Community vote (2/3) + Security Council confirmation |
| **Emergency** | Security Council |

## Special Cases

### Emergency: Reporting a Vulnerability

If you discover a vulnerability:

1. **Do not publish vulnerability details in open access.**
2. Report through the Security Council's secure channel (contact details are published in the project channel).
3. The Security Council will confirm receipt and determine severity.
4. After the vulnerability is patched -- a post-mortem is published.

We value responsible disclosure and plan to launch a bug bounty program.

### Governance Proposals

Changes to the governance process itself (thresholds, deadlines, role composition) go through as **Major** changes with a community vote.

### Treasury Spending Proposals

Proposals for protocol treasury expenditures:

1. Formatted as an RFC with sections: amount, recipient, purpose, milestone schedule.
2. Go through standard review.
3. In Phase 1: approved by maintainers + Security Council.
4. In Phase 2: approved through on-chain DAO vote.

## Code of Conduct

- Constructive criticism is welcomed.
- Support your position with technical facts.
- Respect reviewers' time -- prepare complete RFCs.
- Do not duplicate existing proposals; join the discussion instead.

## Governance Roles

| Role | Description |
|---|---|
| **Contributor** | Any participant proposing an RFC |
| **Core Maintainer** | Responsible for reviewing and accepting Patch/Minor changes |
| **Security Council** | Responsible for Emergency changes and protocol security |
| **Voter** | Ecosystem participant with voting rights (trust_score >= threshold) |

## Contacts

- Official channel: https://t.me/meetagent
- Discussions and proposals: in the channel messages.

---

Document version: 1.0  
Date: 2026-02-14
