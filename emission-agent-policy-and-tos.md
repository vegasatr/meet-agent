# Meet Agent Protocol — Emission Agent Policy & Disclaimer

This document constitutes the **system_agent_policy_ref** for the Emission Agent of the Meet Agent Protocol (the “Protocol”) and defines its operational rules, legal status, and liability limitations.

The Protocol is an open-source software project and is provided on an “AS IS” basis.

---

## 1. Legal Status

**1.1.** The Emission Agent is a software component of the Protocol and does not constitute:

- a financial institution,
- an investment service provider,
- a payment service,
- a broker, dealer, or securities issuer.

**1.2.** Use of the Protocol is voluntary and at the user’s own risk.

**1.3.** Nothing in this document constitutes:

- an offer,
- an investment recommendation,
- financial advice,
- legal or tax advice.

## 2. Role of the Emission Agent

The Emission Agent provides baseline MEET liquidity only when valid peer-to-peer (P2P) offers are insufficient to satisfy buy demand.

The Emission Agent:

- does not compete with user agents when sufficient market volume exists,
- operates strictly according to algorithmic rules,
- does not exercise discretionary decision-making.

## 3. Conditions for Selling MEET

The Emission Agent sells MEET only when all of the following conditions are met:

- valid P2P offers do not cover buy demand;
- the sale price is calculated as: **max(1.00, last_trade_price × 1.01)** and shall not be lower than 1 USDT per 1 MEET;
- the volume sold is limited to the remaining shortfall after P2P matching.

## 4. Conditions for Buyback (Withdraw USDT)

Upon a user request “Withdraw USDT”:

**4.1.** For the first 24 hours, only P2P matching is applied.

**4.2.** After 24 hours, the Emission Agent purchases the remaining MEET amount at the nominal R/C rate.

**4.3.** USDT is transferred from the project treasury to the user’s specified TON wallet.

**4.4.** Target service level (SLA): within 1 hour after the 24-hour window expires.

## 5. Risk Invariants

The Emission Agent operates under the following constraints:

- no sale below the emission price floor;
- period-based emission volume limits;
- rate limits on buyback operations;
- emergency pause mechanism in case of critical events.

## 6. Transparency and Audit

The following data is publicly visible:

- Emission Agent status,
- last_trade_price,
- P2P versus emission volume,
- number of buyback operations and average SLA.

All operations follow the protocol lifecycle: **discovery → Intent → Quote → Deal → settlement.**

Audit records are maintained via system_agent_audit_log.

## 7. Scope (MVP)

- USDT/MEET market only;
- single project treasury;
- manual emergency pause via operator bot.

## 8. Disclaimer of Warranties

The Protocol and Emission Agent are provided “AS IS” and “AS AVAILABLE”, without warranties of any kind, including but not limited to:

- merchantability,
- fitness for a particular purpose,
- uninterrupted operation,
- liquidity,
- accuracy or absence of defects.

## 9. Limitation of Liability

To the maximum extent permitted by law, the developers, contributors, and operators of the Protocol shall not be liable for:

- any direct or indirect financial losses,
- loss of digital assets,
- data corruption or service interruption,
- actions of third-party users or agents,
- blockchain failures or external infrastructure issues,
- regulatory or legal consequences of use.

Users assume full responsibility for assessing their own risk tolerance.

## 10. Regulatory Notice

MEET does not grant:

- governance rights,
- ownership interests,
- profit-sharing rights,
- guaranteed redemption value.

The Protocol is not intended to circumvent applicable laws and must be used in compliance with the user’s local jurisdiction.

## 11. Amendments

This policy may be modified at any time by publishing an updated version in the Protocol repository without prior notice.

---

# Meet Agent Protocol — Terms of Service

**Last updated:** 2026-02-20

By accessing or using the Meet Agent Protocol (the “Protocol”), you agree to be bound by these Terms of Service (“Terms”).

If you do not agree to these Terms, you must not use the Protocol.

## 1. Nature of the Service

**1.1.** The Protocol is an open-source decentralized software system enabling interactions between autonomous agents and users.

**1.2.** The Protocol does not provide custodial services, financial services, or investment products.

**1.3.** Users retain full control over their wallets and assets at all times.

## 2. Eligibility

You represent that:

- you are legally permitted to use blockchain-based software in your jurisdiction;
- you understand the risks of digital assets and smart contracts.

## 3. User Responsibilities

You agree that you:

- use the Protocol at your own risk;
- are solely responsible for your wallet security;
- verify all transaction details before execution;
- comply with applicable laws and regulations.

## 4. No Investment Advice

Nothing in the Protocol or its documentation constitutes:

- financial advice,
- investment solicitation,
- recommendation to buy or sell any asset.

All information is provided for technical and experimental purposes only.

## 5. Risks Disclosure

You acknowledge and accept the risks including but not limited to:

- smart contract vulnerabilities,
- market volatility,
- liquidity shortages,
- transaction delays or failures,
- irreversible blockchain transactions.

## 6. No Guarantees

The Protocol does not guarantee:

- availability or uptime,
- asset value or liquidity,
- execution speed,
- correctness of outputs.

## 7. Limitation of Liability

To the maximum extent permitted by law, the Protocol developers, maintainers, and contributors shall not be liable for:

- any loss of funds,
- lost profits or business opportunities,
- indirect or consequential damages.

## 8. Open Source License

The Protocol is distributed under its applicable open-source license. Use of the software is governed by that license in addition to these Terms.

## 9. Modifications

These Terms may be updated at any time without prior notice. Continued use of the Protocol constitutes acceptance of the updated Terms.

## 10. Governing Law

These Terms shall be governed by and construed in accordance with the laws applicable in the user’s jurisdiction, unless otherwise required by mandatory law.

## 11. Contact

The Protocol is maintained as an open-source project. No warranties or support obligations are implied unless explicitly stated.
