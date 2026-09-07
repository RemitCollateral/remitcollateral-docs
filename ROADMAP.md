# Roadmap

Where RemitCollateral is, and what stands between here and a protocol that can
hold real money.

**There are no dates here.** Items are ordered by dependency, not by calendar —
each phase mostly needs the one before it. Anything that looks like a commitment
to a date would be invented, and this document is more useful as an honest
account of what is actually left to do.

The authoritative record of what is currently wired versus mocked lives in
[ARCHITECTURE.md](ARCHITECTURE.md#integration-status). When an item here ships,
that section should shrink.

---

## Phase 0 — where we are

v1 is feature-complete at each layer and mocked between two of them.

- ✅ Three Soroban contracts — `GuarantorVault`, `LoanLedger`,
  `LiquidationEngine` — with test suites and recorded ledger snapshots.
- ✅ The full `/api/v1` surface: auth, guarantors, vaults, beneficiaries, loans,
  repayments, remittances, audit, admin.
- ✅ The reputation engine, the loan lifecycle sweep, and the audit trail.
- ✅ The guarantor dashboard, running against either the mock or the live API.
- ✅ Protocol documentation, license, and security policy.

What this does **not** yet include: a single transaction against a real ledger,
a single real disbursement, or any state that survives a restart.

---

## Phase 1 — reconcile the repositories

Small, cheap, and blocking. The three repositories were built against a shared
design but have drifted at four seams, documented under
[known cross-repository mismatches](ARCHITECTURE.md#known-cross-repository-mismatches).
Until these agree, `live` mode does not work end to end.

- [ ] **Align the auth header.** The frontend sends `Authorization: Bearer`; the
  backend reads `x-wallet-address`. Pick one — ideally the bearer token, since
  that is where SEP-10 lands in Phase 2 — and change the other.
- [ ] **Align the default API port.** The frontend defaults to `3001`, the
  backend serves `4000`.
- [ ] **Reconcile the grace period.** The contract deployment guidance suggests
  14 days; the backend and frontend both default to 7. This one matters most:
  the chain and the lifecycle sweep would otherwise disagree about the moment a
  loan defaults, which is a disagreement about money.
- [ ] **Add `GET /beneficiaries`.** The frontend currently reconstructs the list
  from the dashboard payload because no list endpoint exists.

Doing this first means every later phase is tested against a stack that actually
talks to itself.

---

## Phase 2 — testnet

The point of this phase is one real loan, originated and repaid end to end,
against deployed contracts on Stellar testnet.

- [ ] **Live `ContractGateway`.** The largest single piece of work in the
  roadmap: a Stellar SDK implementation of the existing interface, invoking the
  deployed contracts. Because the interface already exists and every service
  goes through it, nothing in `src/services` should need to change — that is the
  design being cashed in.
- [ ] **Deploy and wire the contracts** on testnet, following the
  [deployment sequence](ARCHITECTURE.md#deployment-and-wiring). The wiring steps
  fail quietly when skipped, so this wants a scripted, repeatable deployment
  rather than manual CLI invocations.
- [ ] **SEP-10 authentication.** Replace the `x-wallet-address` header stub with
  real challenge-response signature verification. The header must never reach a
  deployed environment.
- [ ] **Persistent storage.** PostgreSQL behind the stores. The `pg` dependency
  is already present and the stores are structured for the port.
- [ ] **FX rate oracle.** Origination currently sets `principalUsd =
  principalLocal`. Local-currency loans cannot price correctly until this is
  real.
- [ ] **Reconcile chain and backend state.** Both track loan status and
  collateral. Decide which is authoritative on disagreement — the chain, almost
  certainly — and build the reconciliation that detects drift.

---

## Phase 3 — first partner integration

Everything above can be done without leaving the building. This phase cannot: it
needs a licensed off-ramp partner willing to disburse and collect.

- [ ] **A real `OffRampAdapter`.** One implementation of the existing interface
  per partner. `MockOffRampAdapter` defines the shape.
- [ ] **Attestation signature verification.** `verifyAttestation` must actually
  verify a partner signature rather than accept the call. This is the hinge the
  whole trust model turns on — a repayment is only ever as trustworthy as this
  check.
- [ ] **Partner onboarding and key management.** Registering a partner address
  on-chain, issuing and rotating API credentials, and exercising revocation.
- [ ] **Remittance history ingest** from the partner's records, which is what
  makes the cold-start reputation signal real rather than seeded.
- [ ] **Beneficiary notifications.** The beneficiary has no wallet and no
  dashboard, so SMS is the only channel through which they learn a schedule
  exists and that a payment is due.
- [ ] **Disbursement failure handling** against a real partner — timeouts,
  partial failures, and reconciling a disbursement whose outcome is unknown.
  The origination rollback assumes a clean throw; reality is messier.

---

## Phase 4 — production hardening

Nothing in this phase adds a feature. All of it is what separates a working
prototype from something that can hold a stranger's collateral.

- [ ] **A third-party security audit of the contracts.** Non-negotiable before
  mainnet. The [security policy](SECURITY.md) states plainly that the contracts
  are unaudited.
- [ ] **TTL management.** Soroban archives state that is not refreshed. Vault and
  loan entries are long-lived by nature, so their lifetime has to be extended
  deliberately or a dormant loan becomes unreadable.
- [ ] **Multi-instance operation.** The lifecycle sweep runs in-process and
  assumes a single backend. Running several would sweep several times per tick,
  so this needs an external scheduler or a distributed lock.
- [ ] **Admin and oracle key management.** Both are single keys today. Multisig
  or a hardware-backed signer, plus a documented rotation procedure.
- [ ] **Monitoring and alerting.** Sweep failures, gateway errors, and
  disbursement failures are currently visible only in the audit log. A default
  that goes unrecorded is a silent loss.
- [ ] **Load and failure testing**, particularly around the sweep with a large
  book of open loans.

---

## Phase 5 — protocol evolution

These change what the protocol *is*, not just how well it runs. Each reduces a
trust assumption that v1 accepts deliberately. They are listed roughly by how
much trust they remove.

**Multi-partner attestation.** Today a single registered partner can attest a
repayment. Requiring corroborating attestations from more than one partner
before a repayment counts removes the largest remaining single point of trust.

**On-chain reputation derivation.** The chain currently trusts an oracle's
published score, and that score sets the LTV. Moving part of the derivation
on-chain makes the LTV reproducible without trusting the oracle — the most
valuable of these items, and the hardest, since the scoring inputs are
themselves off-chain.

**DEX-based liquidation.** Forfeited collateral currently transfers to a
platform-controlled settlement address. Settling through a swap on the Stellar
DEX removes the platform as a custodian of seized funds.

**Per-loan grace configuration.** The grace period is one global value. Letting
it vary with loan size or beneficiary reputation would let a trusted borrower
absorb a late payment without the same consequence as a first-time one.

**Guarantor-side risk controls.** Exposure limits per beneficiary, and a way for
a guarantor to decline further origination against their vault without
withdrawing.

---

## Explicitly not planned

Stating these saves the argument later.

- **A beneficiary wallet or app.** The beneficiary never touching crypto is the
  premise of the protocol, not a limitation of the current version.
- **Pooled collateral.** Vault isolation is a core invariant. A shared pool would
  socialise losses across unrelated guarantors and is not on the table.
- **Accepting repayment claims from the beneficiary or guarantor.** Repayments
  come from a registered partner or they do not count.
- **Putting beneficiary identity on-chain.** The 32-byte handle exists precisely
  so personal data never reaches a public ledger.

---

## Contributing to the roadmap

If you want to work on something here, open an issue in
`remitcollateral-docs` first so the design can be agreed across the layers it
touches — most of these items span more than one repository. See
[cross-repository changes](CONTRIBUTING.md#cross-repository-changes) for how to
sequence the work so `main` is never broken in either place.
