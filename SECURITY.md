# Security policy

RemitCollateral custodies real collateral on behalf of guarantors and authorizes
the release of real money to people who cannot see the chain. We take reports
seriously and would rather hear a false alarm than miss a genuine issue.

## Project status

> **This is pre-production software. The contracts have not been audited.**

The contracts are deployed on Stellar testnet with a 2-of-3 multisig admin and a
48-hour timelock on upgrades and settlement changes. The backend connects to
them for sign-in, deposits, withdrawals and loan origination. Repayments and
liquidation still run only in the backend's own records, the off-ramp partner is
a mock, and persistence is in-memory. Do not deploy it against mainnet funds or
real beneficiary money.
The [integration status](ARCHITECTURE.md#integration-status) section of the
architecture document records exactly what is wired and what is stubbed.

## Scope

This policy covers all four repositories:

| Repository | Surface |
|------------|---------|
| `remitcollateral-contract` | Soroban contracts — collateral custody, loan state, liquidation |
| `remitcollateral-backend` | API, reputation engine, off-ramp adapters, lifecycle sweep |
| `remitcollateral-frontend` | Guarantor dashboard, wallet connection |
| `remitcollateral-docs` | Protocol design — a flaw in the *design* is in scope |

A design flaw documented here or in [ARCHITECTURE.md](ARCHITECTURE.md) is a valid
report even if no code implements it incorrectly. If the protocol as specified is
exploitable, we want to know before it ships.

## Reporting a vulnerability

**Do not open a public issue.**

Report privately through GitHub: go to the affected repository's **Security** tab
and choose **Report a vulnerability**, which opens a private advisory visible
only to the maintainers. If you cannot use that channel, contact a maintainer
privately and ask for a secure channel before sending any detail.

Please include what you can:

- Which repository and component (contract function, route, middleware, screen).
- Steps to reproduce, or a failing test if you have one.
- Impact — specifically, what an attacker gains or what a victim loses.
- Any suggested remediation.

We aim to acknowledge a report within **3 business days**, and to give you an
assessment of severity and a rough fix timeline within **10 business days**.

## What we consider severe

Because this is a custody protocol, severity tracks the money and the trust
boundaries rather than the usual web categories. The following are the highest
priority, roughly in order:

**Critical**

- Moving collateral out of a vault without the guarantor's authorization.
- Forfeiting collateral without a genuine, expired grace period.
- Cross-vault contamination — any path by which one guarantor's loss touches
  another guarantor's balance. Vaults are isolated by design and this must hold
  absolutely.
- Bypassing the co-signed attestation, so a repayment is credited without both
  the loan's own partner and a registered verifier signing it.
- Releasing more collateral than the repayment schedule earns, or closing a loan
  without the principal actually being repaid.

**High**

- Manipulating a beneficiary's reputation score, and therefore the required LTV,
  from an unprivileged position — including writing `partner_reported`
  remittances without partner credentials.
- Attributing an attestation to a partner that did not make it.
- Originating a loan with less collateral locked than the computed LTV requires,
  or with no lock at all.
- Privilege escalation to the admin, oracle or verifier role, or running an
  upgrade or settlement change without waiting out the timelock.
- Blocking the permissionless liquidation cranks, so defaults cannot be recorded.

**Medium and below**

- Leaking beneficiary personal data — phone numbers, KYC references — into
  on-chain state or logs. The beneficiary handle is a keyed 32-byte digest
  precisely so this cannot happen; a path that defeats it is a real issue.
- Denial of service against the API or the lifecycle sweep.
- Audit trail gaps that make a state change unreconstructable.

The protocol invariants listed in
[CONTRIBUTING.md](CONTRIBUTING.md#protocol-invariants) are a good checklist. If
you find a way to break one, that is very likely a valid report.

## Known limitations — already documented

These are real weaknesses, but they are known and recorded in
[ARCHITECTURE.md](ARCHITECTURE.md#pending). You are welcome to report them, but
they are already tracked and will not be treated as new findings:

- **Repayments and liquidation are not on chain yet.** With the contracts
  connected, an attestation updates only the backend's records and releases no
  collateral on chain, and the lifecycle sweep does not drive the liquidation
  cranks.
- **A failed payout cannot be undone on chain.** Collateral locked for a loan
  whose disbursement then fails stays locked; the backend records
  `LOAN_DISBURSEMENT_FAILED` for an operator to resolve.
- **The off-ramp adapter is a mock.** Its `verifyAttestation` accepts any
  non-empty signature, and its exchange rates are fixed.
- **Persistence is in-memory.** State, sessions included, does not survive a
  restart.
- **A partner and the verifier together are trusted.** Both signatures are
  required for a repayment, but if both collude they can credit one that never
  happened. A dispute window before released collateral becomes withdrawable is
  on the roadmap.
- **The oracle and verifier are single keys** held by the backend. Only the admin
  is a multisig.
- **Most admin powers take effect immediately.** Only upgrades and settlement
  changes wait out the timelock; registering or revoking partners and verifiers,
  reassigning a loan's partner and setting the oracle do not.
- **Reputation derivation is off-chain.** The on-chain LTV trusts the oracle's
  published score. Moving part of the derivation on-chain is on the roadmap.
- **The lifecycle sweep assumes a single instance.** Running several backends
  would run the sweep more than once per tick.

A *new* way to exploit one of these beyond its documented effect is still worth
reporting.

## Out of scope

- Findings that require a compromised guarantor wallet, enough of the admin
  council's keys to meet its threshold, the oracle key, or both the partner's and
  the verifier's keys — those roles are trusted by construction.
- Vulnerabilities in Freighter, the Stellar network, Soroban itself, or a
  third-party off-ramp partner's own systems. Report those upstream.
- Automated scanner output with no demonstrated impact.
- Missing security headers or best-practice recommendations with no exploit path.
- Social engineering of maintainers or users.

## Disclosure

We ask for coordinated disclosure. Please give us a reasonable window to ship a
fix before publishing — **90 days** is our default, and we will usually be much
faster than that for anything affecting custody. If a fix is taking longer, we
will tell you why rather than let the clock run out silently.

We are happy to credit you in the advisory and the release notes. Tell us how you
would like to be named, or if you would rather stay anonymous.

## Safe harbour

We will not pursue or support legal action against anyone who makes a good-faith
effort to follow this policy: who reports promptly and privately, who does not
access, modify, or destroy data belonging to others, who tests only against
testnet or their own accounts, and who avoids degrading the service for other
users.

If you are unsure whether something is in scope or whether your testing would
cross a line, ask first.
