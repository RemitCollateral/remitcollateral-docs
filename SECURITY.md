# Security policy

RemitCollateral custodies real collateral on behalf of guarantors and authorizes
the release of real money to people who cannot see the chain. We take reports
seriously and would rather hear a false alarm than miss a genuine issue.

## Project status

> **This is pre-production software. The contracts have not been audited.**

v1 runs on testnet with mocked integrations at two boundaries — the Soroban
contract gateway and the off-ramp partner adapter — and with in-memory
persistence. Do not deploy it against mainnet funds or real beneficiary money.
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
- Bypassing the partner attestation requirement, so a repayment is credited
  without a registered partner authorizing it.
- Releasing more collateral than the repayment schedule earns, or closing a loan
  without the principal actually being repaid.

**High**

- Manipulating a beneficiary's reputation score, and therefore the required LTV,
  from an unprivileged position — including writing `partner_reported`
  remittances without partner credentials.
- Attributing an attestation to a partner that did not make it.
- Originating a loan with less collateral locked than the computed LTV requires,
  or with no lock at all.
- Privilege escalation to the admin or oracle role.
- Blocking the permissionless liquidation cranks, so defaults cannot be recorded.

**Medium and below**

- Leaking beneficiary personal data — phone numbers, KYC references — into
  on-chain state or logs. The beneficiary handle is a 32-byte digest precisely so
  this cannot happen; a path that defeats it is a real issue.
- Denial of service against the API or the lifecycle sweep.
- Audit trail gaps that make a state change unreconstructable.

The protocol invariants listed in
[CONTRIBUTING.md](CONTRIBUTING.md#protocol-invariants) are a good checklist. If
you find a way to break one, that is very likely a valid report.

## Known limitations — already documented

These are real weaknesses, but they are known, deliberate to v1, and recorded in
[ARCHITECTURE.md](ARCHITECTURE.md#stubbed-or-pending). You are welcome to report
them, but they are already tracked and will not be treated as new findings:

- **Wallet auth is a header stub.** The backend reads `x-wallet-address` and
  accepts any non-empty signature rather than verifying SEP-10. It must not be
  treated as authentication in any deployed environment.
- **The contract gateway and off-ramp adapter are mocks.** No real chain calls or
  partner calls happen in v1.
- **Persistence is in-memory.** State does not survive a restart.
- **FX is hardcoded 1:1.** Local-currency loans do not price correctly.
- **Reputation derivation is off-chain.** The on-chain LTV trusts the oracle's
  published score. Moving part of the derivation on-chain is on the roadmap.
- **The lifecycle sweep assumes a single instance.** Running several backends
  would run the sweep more than once per tick.

A *new* way to exploit one of these beyond its documented effect is still worth
reporting.

## Out of scope

- Findings that require a compromised guarantor wallet, admin key, or oracle key
  — those roles are trusted by construction.
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
